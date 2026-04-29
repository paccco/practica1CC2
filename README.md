# Práctica 1: Despliegue de Servicio OwnCloud con Alta Disponibilidad

**Autor:** Francisco de Asís Carrasco Conde  
**Entorno de Desarrollo:** Podman en modo Rootless  
**Arquitectura:** Escenario 2 (Empresa Mediana - Alta Disponibilidad)

---

## 1. Descripción del Proyecto
Esta práctica consiste en el diseño y despliegue de un servicio de almacenamiento en la nube basado en **OwnCloud**, utilizando una arquitectura de microservicios contenerizados con **Podman**.

Siguiendo los requisitos del **Escenario 2**, se ha implementado una infraestructura robusta que garantiza la alta disponibilidad mediante la replicación de servicios y el balanceo de carga dinámico.

## 2. Arquitectura del Sistema
La solución se compone de los siguientes subservicios interconectados:

* **Balanceador de Carga (HAProxy):** Actúa como punto de entrada único, distribuyendo el tráfico entre las réplicas de OwnCloud mediante un algoritmo *round-robin*.
* **Servicio Web (OwnCloud):** Implementado con dos réplicas estáticas para asegurar la continuidad del servicio ante caídas de nodos.
* **Base de Datos (MariaDB):** SGBD relacional para la persistencia de metadatos, configurado con volúmenes permanentes.
* **Caché (Redis):** Servicio de memoria caché para optimizar el bloqueo de ficheros y el rendimiento general.
* **Autenticación (LDAP):** Servidor OpenLDAP para la gestión centralizada de usuarios y grupos.

## 3. Configuración de los Servicios

### 3.1. Replicación y Alta Disponibilidad
Para cumplir con el objetivo de alta disponibilidad, se han definido dos servicios de aplicación independientes (`owncloud1` y `owncloud2`) que comparten el mismo almacenamiento persistente.

### 3.2. Balanceo de Carga con HAProxy
Se utiliza una imagen de `haproxy:2.4` configurada para monitorizar la salud de los contenedores de OwnCloud.

* **Puerto de Servicio:** `${PORT_OWNCLOUD}` (Redirigido al puerto 80 de las réplicas).
* **Puerto de Estadísticas:** `8404` (Dashboard de monitorización en tiempo real).

### 3.3. Persistencia de Datos
Se han implementado volúmenes de datos con el sufijo `:z` para garantizar la compatibilidad con los contextos de seguridad de **SELinux** en el servidor de producción:

* **MariaDB:** `./mariadb/data:/var/lib/mysql:z`
* **LDAP:** `./ldap/db:/var/lib/ldap:z` y `./ldap/config:/etc/ldap/slapd.d:z`

## 4. Instrucciones de Despliegue

1.  **Configurar variables de entorno:** Cree un archivo `.env` basado en `.env.example` con sus puertos asignados y contraseñas.
2.  **Lanzar la infraestructura:** Ejecute el siguiente comando para levantar todos los servicios en segundo plano:

```bash
podman-compose up -d
