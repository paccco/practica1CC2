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
