# Objetivo
En este documento se indican los pasos necesarios para montar un Lab de Marquez configurandolo de tal manera que se utilice una base de datos PostgreSQL externa, al igual que usar una instancia de OpenSearch externa. Por defecto, si se siguen los pasos del manual de Marquez, lo que ofrece es un script que levanta todos los servicios dentro de un Docker Compose, pero para utilizar componentes que ya tenemos para otros proyectos como es una instancia de PostgreSQL, hay que hacer una serie de cambios.

# Componentes de Marquez

- API
- Web
- PostgreSQL para metadatos
- OpenSearch para indexación utilizada en las búsquedas en la web.

# Pasos para montar el Lab

## Montar PostgreSQL y OpenSearch

Por si no tenéis estos servicios corriendo para otros temas, os he dejado en el directorio **OpenSearch** y el directorio **PostgreSQL** una configuración base para lanzar cada uno de estos servicios con Docker Compose. Para que no dé errores al arrancar estos servicios, se deben borrar antes de lanzar, los archivos **.keep** que hay en los directorios **OpenSearch/opensearch_data** y en **PostgreSQL/postgres_data**

## Crear base de datos Marquez
En PostgreSQL se debe crear la base de datos que usará Marquez para sus metadatos. Para crear la base de datos y el usuario:

```sql
CREATE DATABASE marquez_00;
CREATE USER marquez WITH PASSWORD 'XXXXXXXXXXXX';
```
Se dan permisos al usuario:

```sql
GRANT ALL PRIVILEGES ON DATABASE marquez_00 TO marquez;
```
Y conectados a la base de datos que se ha creado, se dan permisos sobre el esquema al usuario. Esta configuración se debe hacer a partir de la versión 15 de PostgreSQL:

```sql
GRANT ALL ON SCHEMA public TO marquez;
```

## Descargar Proyecto Marquez

Para obtener el código de Marquez, descargaremos el código del proyecto desde el respositoio github, de la siguiente manera:

```bash
git clone https://github.com/MarquezProject/marquez && cd marquez
```

## Configuración
Para configurar el servicio de Marquez apuntando a nuestra PostgreSQL e instancia de OpenSearch tenemos que crear imágenes custom de los servicios de **API** y **Web**.

### Crear imagen custom API
Lo primero que debemos hacer es crear una copia de dos archivos:
- **marquez.dev.yml** en el raíz del directorio del proyecto.
- **entrypoint.sh** en el direcotrio docker del proyecto.


```bash
cp marquez.dev.yml marquez.yml
cd docker
cp entrypoint.sh entrypoint_custom.sh
cd ..
```
Se crea el archivo Dockerfile_custom con el siguiente contenido:

```dockerfile
FROM eclipse-temurin:17 AS base
WORKDIR /usr/src/app
COPY gradle gradle
COPY gradle.properties gradle.properties
COPY gradlew gradlew
COPY settings.gradle settings.gradle
RUN ./gradlew --version

FROM base AS build
WORKDIR /usr/src/app
COPY build.gradle build.gradle
COPY api ./api
COPY clients/java ./clients/java
RUN ./gradlew --no-daemon clean :api:shadowJar

FROM eclipse-temurin:17
RUN apt-get update && apt-get install -y postgresql-client bash coreutils
WORKDIR /usr/src/app
COPY --from=build /usr/src/app/api/build/libs/marquez-*.jar /usr/src/app
COPY marquez.yml marquez.yml
COPY docker/entrypoint_custom.sh entrypoint.sh
EXPOSE 5000 5001
ENTRYPOINT ["/usr/src/app/entrypoint.sh"]
```

El contenido es el mismo que el del archivo **Dockerfile** del directorio raíz, modificando la referencia al archivo de configuración por **marquez.yml** y el de entrypoint a **entrypoint_custom.sh**

Se edita el archivo **entrypoint_custom.sh** con el siguiente contenido:
```bash
#!/bin/bash
#
# Copyright 2018-2023 contributors to the Marquez project
# SPDX-License-Identifier: Apache-2.0
#
# Usage: $ ./entrypoint.sh

set -e

if [[ -z "${MARQUEZ_CONFIG}" ]]; then
  MARQUEZ_CONFIG='marquez.yml'
  echo "WARNING 'MARQUEZ_CONFIG' not set, using development configuration."
fi

# Adjust java options for the http server
JAVA_OPTS="${JAVA_OPTS} -Duser.timezone=UTC -Dlog4j2.formatMsgNoLookups=true"

# Start http server with java options and configuration
java ${JAVA_OPTS} -jar marquez-*.jar server ${MARQUEZ_CONFIG}
```

En donde el cambio es configurar para que utilice el archivo **marquez.yml** en lugar del **marquez.dev.yml**. 

Y ahora por último, editamos el archivo marquez.yml que hemos copiado en el directorio raíz, con el contenido similar al siguiente:
```
server:
  applicationConnectors:
  - type: http
    port: 5000
    httpCompliance: RFC7230_LEGACY
  adminConnectors:
  - type: http
    port: 5001

db:
  driverClass: org.postgresql.Driver
  url: jdbc:postgresql://xxx.xxx.xxx.xxx:5432/marquez
  user: marquez
  password: XXXXXXXXXXXX

migrateOnStartup: true

graphql:
  enabled: true

logging:
  level: INFO
  appenders:
    - type: console

search:
  enabled: true
  scheme: http
  host: xxx.xxx.xxx.xxx
  port: 9200
  username: admin
  password: XXXXXXXXXXXX

tags:
  - name: PII
    description: Personally identifiable information
  - name: SENSITIVE
    description: Contains sensitive information
```

En este archivo se incluyen la información de conexión a PostgreSQL, la conexión a OpenSearch y los puertos que se usarán para el servicio de la API.


Y para crear la imagen custom, lanzamos:

```bash
docker build -f Dockerfile_custom -t marquez:0.50.0_custom --no-cache --rm .
```

### Crear imagen custom para la Web
Dentro del directorio **web** creamos un archivo Dockerfile_custom con el siguiente contenido:
```dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app

RUN apk update && apk add --virtual bash coreutils
RUN apk add --no-cache git
COPY package*.json ./

RUN npm install
COPY . .
RUN REACT_APP_ADVANCED_SEARCH=true npm run build
COPY docker/entrypoint.sh entrypoint.sh
EXPOSE 3000
ENTRYPOINT ["/usr/src/app/entrypoint.sh"]
```

Se ha creado a partir del archivo **Dockerfile**. Para crear la imagen custo, lanzaremos:

```bash
docker build -f Dockerfile_custom -t marquez:0.50.0_custom --no-cache --rm .
```

## Lanzar Marquez

Se debe crear en el directorio **marquez_data** que se mapeará con el directorio **/opt/marquez** en el backend y se podrá lanzar el servicio. Para ello se incluye un arqchivo **docker-compose.yaml** que permite lanzar Marquez apuntando a PostgreSQL y OpenSearch con el comando:

```bash
docker-compose up -d 
```

