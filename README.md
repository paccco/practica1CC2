# Práctica 1: Despliegue de Servicio OwnCloud con Alta Disponibilidad

**Autor:** Francisco de Asís Carrasco Conde  
**Entorno de Desarrollo:** Podman en modo Rootless  
**Arquitectura:** Escenario 2 (Empresa Mediana - Alta Disponibilidad)

---

## 1. Descripción del Proyecto
Esta práctica consiste en el diseño y despliegue de un servicio en la nube basado en **OwnCloud**, utilizando una arquitectura de microservicios en contenedores con **Podman**.

Siguiendo los requisitos del **Escenario 2**, se ha implementado una infraestructura robusta que garantiza la alta disponibilidad replicando servicios y el balanceo de carga dinámico.

## 2. Arquitectura del Sistema
La solución se compone de los siguientes subservicios interconectados:

* **Balanceador de Carga (HAProxy):** Actúa como punto de entrada único, distribuyendo el tráfico entre las réplicas de OwnCloud.
* **Servicio Web (OwnCloud):** Implementado con dos réplicas estáticas.
* **Base de Datos (MariaDB):** SGBD relacional para la persistencia de metadatos, configurado con volúmenes.
* **Caché (Redis):** Servicio de memoria caché.
* **Autenticación (LDAP):** Servidor OpenLDAP.

## 3. Configuración de los Servicios

### 3.1. Replicación y Alta Disponibilidad
Se han definido dos servicios de aplicación independientes (`owncloud1` y `owncloud2`) que comparten el mismo volumen.

### 3.2. Balanceo de Carga con HAProxy
Se utiliza la imagen `haproxy:2.4` configurada para monitorizar la salud de los contenedores OwnCloud.

* **Puerto de Servicio:** `${PORT_OWNCLOUD}` (Redirigido al puerto 80 de las réplicas).
* **Puerto de Estadísticas:** `8404` (Dashboard de monitorización en tiempo real).

### 3.3. Persistencia de Datos
Se han implementado volúmenes de datos con el sufijo `:z` por temas de permisos en contenedores podman:

* **MariaDB:** `./mariadb/data:/var/lib/mysql:z`
* **LDAP:** `./ldap/db:/var/lib/ldap:z` y `./ldap/config:/etc/ldap/slapd.d:z`

## 4. Instrucciones de Despliegue

1.  **Configurar variables de entorno:** Cree un archivo `.env` basado en `.env.example` con puertos asignados y contraseñas.
```
# Rango de puertos asignado (20011 - 20020)
PORT_OWNCLOUD=20011
PORT_LDAP=20012
PORT_LDAP_SSL=20013
PORT_MARIADB=20014
PORT_STATS=20015

# Secretos de Base de Datos
DB_ROOT_PASS=root_secure_pass
DB_USER=owncloud_db_user
DB_PASS=owncloud_secure_pass
DB_NAME=owncloud_db

# Secretos de LDAP
LDAP_ADMIN_PASS=admin_ldap_pass
LDAP_DOMAIN=example.org
LDAP_ORG="empresa"

# Secretos owncloud
# --- OJO se que esto no es seguro pero lo hago
# porque estamos en una práctica ------
OWNCLOUD_TRUSTED_DOMAINS=*
ADMIN_USERNAME=admin
ADMIN_PASSWORD=admin
```
3.  **Lanzar la infraestructura:** Ejecutar el siguiente comando para levantar todos los servicios:

```bash
podman-compose up -d
```

## Configuración del Stack (docker-compose.yml)
### SGBD(MariaDB)
```yaml
mariadb:
  image: mariadb:10.6
  container_name: mariadb_server
  environment:
    - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
    - MYSQL_DATABASE=owncloud
    - MYSQL_USER=owncloud
    - MYSQL_PASSWORD=${DB_PASSWORD}
  volumes:
    - ./mariadb/data:/var/lib/mysql:z
  restart: always
```
### LDAP
```yaml
ldap:
  image: osixia/openldap:1.5.0
  container_name: ldap_server
  environment:
    - LDAP_ORGANISATION=UGR
    - LDAP_DOMAIN=ugr.es
    - LDAP_ADMIN_PASSWORD=${LDAP_ADMIN_PASSWORD}
  volumes:
    - ./ldap/db:/var/lib/ldap:z
    - ./ldap/config:/etc/ldap/slapd.d:z
```
### Owncloud
```yaml
owncloud1:
    image: owncloud:10.0.10
    container_name: oc_replica_1
    restart: always
    depends_on:
      - db
      - redis
      - ldap
    environment: &oc-env #Esto es un alias
      - OWNCLOUD_DB_TYPE=mysql
      - OWNCLOUD_DB_NAME=${DB_NAME}
      - OWNCLOUD_DB_USERNAME=${DB_USER}
      - OWNCLOUD_DB_PASSWORD=${DB_PASS}
      - OWNCLOUD_DB_HOST=db
      - OWNCLOUD_REDIS_ENABLED=true
      - OWNCLOUD_REDIS_HOST=redis
      - OWNCLOUD_TRUSTED_DOMAINS=${OWNCLOUD_TRUSTED_DOMAINS}
      - OWNCLOUD_ADMIN_USERNAME=${ADMIN_USERNAME}
      - OWNCLOUD_ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - OWNCLOUD_SKIP_CHOWN=true
    volumes: &oc-volumes # Igual que para env
      - ./owncloud/data:/var/www/html/data:z
      - ./owncloud/config:/var/www/html/config:z

  owncloud2:
    image: owncloud:10.0.10
    container_name: oc_replica_2
    restart: always
    depends_on:
      - db
      - redis
      - ldap
    environment: *oc-env
    volumes: *oc-volumes
```
Se ha intentado implementar la creacion dinamica de réplicas usando 'duplicate: 2' de esta forma podman levantaría 2 servicios con los mismos parámetros pero generaría nombres automáticos (owncloud_1,owncloud_2,etc), pero surgieron varios problemas con HAProxy por eso se configuró finalmente de forma estática.

Para configurar ldap nos vamos arriba a la izquierda le damos al botón y vamos a la parte dde market. Buscamos la extensión 'LDAP integration' la instalamos. Nos vamos arriba a la derecha donde el nombre de usuario pulsamos para que aparezca el desplegable, se selecciona ajustes. En el menú de la izquierda vamos a administración de usuarios y nos aparecerá el apartado para configurar LDAP. Seguimos estos pasos para el apartado servidor:

1. Ponemos el dominio del servidor (ldap_server en mi caso) y le damos a detectar puerto para rellenarlo automaticamente.
2. DN de usuario: cn=admin,dc=example,dc=org en mi caso.
3. Contraseña: la que tenemos en el .env (admin_ldap_pass en mi caso).
4. Base DN: dc=example,dc=org en mi caso.
5. Probamos conexión y seguimos

Parte de usuarios:

1. Sólo estas clases de objetos: seleccionamos inetOrgPerson para ver las personas creadas en el ldif
2. El filtro es opcionak
3. Comprobamos y seguimos

Parte atributos inicio de sesión, para decirle a ldap que debe comparar para inicio de sesión con los usuarios:

1. Nombre de usuario LDAP/AD: Debe estar marcado
2. Debe quedar así la consulta de abajo (&(|(objectclass=inetOrgPerson))(uid=%uid))

La pestaña grupos para que se vea el grupo creado en mi ldif en el apartado 'Sólo estas clases de objetos' selecionamos 'posixGroup'.

### HAProxy
```yaml
  haproxy:
    image: haproxytech/haproxy-alpine:2.4
    container_name: haproxy_balancer
    restart: always
    ports:
      - "${PORT_OWNCLOUD}:80"
      - "${PORT_STATS}:8404"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro,z
    depends_on:
      - owncloud1
      - owncloud2
```

### Redis

```yaml
redis:
    image: redis:7.2-alpine
    container_name: redis_server
    restart: always
```

4. ## Conclusiones
El despliegue bajo el Escenario 2 garantiza que el servicio de almacenamiento no dependa de un único punto de fallo en la capa de aplicación. La integración con LDAP permite una escalabilidad administrativa adecuada para una empresa mediana, mientras que Redis asegura que la concurrencia de usuarios no degrade el rendimiento del sistema de archivos.

5. ## Bibliografía
LDAP

    https://www.openldap.org/doc/admin26/quickstart.html
    https://computingforgeeks.com/run-openldap-server-in-docker-containers/#google_vignette
    https://github.com/osixia/docker-openldap

HAProxy 2.4

    http://docs.haproxy.org/2.4/intro.html

Owncloud

    https://doc.owncloud.com/server/next/admin_manual/configuration/user/user_auth_ldap.html
