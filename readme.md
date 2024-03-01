# How to set up Docker on a fullstack Project

This readme should allow you to run your 
database, backend, and frontend in individual 
containers using Docker using a single command (hopefully).

* This readme assumes you have your Spring API endpoints, Entity, etc set up!

## The Stack:
- PostgreSQL - postgres:alpine3.19
- Java Spring Boot - 3.2.2
  - Gradle - 8.5 
  - Spring Web
  - Spring JPA
  - PostgreSQL Driver
- Angular - 16.2.0

## Environment
- MacOS Ventura 13.0, Apple M1 Pro chip
- IDE: IntelliJ Ultimate 2022.3.1
  - JDK:17 Corretto
- Docker Desktop (installed Feb 2022)
  - Version 27.2
  - Engine 25.0.3
  - Compose v2.24-5-desktop.1
  - Kubernetes: v1.29.1

# File Structure

Before we begin know that your file structure
may differ. Take that into consideration and if
you get stuck try matching the file structure.

### You will need to create (view file structure below for where to create):
- Dockerfile x2
- .dockerignore x2
- .env (if you don't have it already)
- docker.compose.yml

Note* while creating the Dockerfiles you do not 
need any type of extension like Dockerfile.xyz, 
you just write "Dockerfile".

"..." indicates your other files in your project.

```
 ├── project-name
 │   ├── backend
 │       ├── ...
 │       ├── .dockerignore
 │       └── Dockerfile
 │   └── frontend
 │       ├── ...
 │       ├── .dockerignore
 │       └── Dockerfile
 ├── ...
 ├── .env
 └── docker-compose.yml
```

# Files

Feel free to remove comments if you wish.

## docker-compose.yml
```md
version: '3.9'

services:
  # "database" is semantic, it can be any name
  database:
    image: "postgres:alpine3.19"
    # container_name appears in terminal and in Docker Desktop as the name for the container
    container_name: postgres-database
    # WARNING: I DON'T KNOW HOW TO MANAGE SECURITY
    env_file:
      - .env
    # ports defines what is exposed to the to [local machine]: (from)[docker container]
    ports:
      - "5432:5432"
    # manage database data inline with your project(see note below in this md file)
    volumes:
      - ./database/data:/var/lib/postgresql/data
    # this health check ensures the database files have been completed in the initial spin 
    # up of the database. Subsequent uses the database is already established, and it doesn't matter.
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d postgres"]
      interval: 3s
      retries: 15

  backend:
    container_name: spring-backend
    build:
      context: backend/
    ports:
     - "8080:8080"
    # WARNING: I DON'T KNOW HOW TO MANAGE SECURITY
    env_file:
     - .env
    environment:
      POSTGRES_HOST: postgres
    depends_on:
      database:
        condition: service_healthy

  frontend:
    container_name: angular-frontend
    build:
      context: frontend/
    depends_on:
      database:
        condition: service_healthy
    ports:
      - "4200:80"
```
```md 
# this change would store the database in a 
# volume controlled by Docker (theoretically, I didn't test)
services:
    database:
        volumes:
            - postgres_data:/var/lib/postgresql/data
volumes:
  postgres_data:
```



# application.properties file

Path to this file (created by Spring Initializer): 
 - project-name/backend/src/main/resources/application.properties 

This is where you define username and password for the database!!!
```properties
spring.datasource.username=admin
spring.datasource.password=admin
spring.jpa.hibernate.ddl-auto=create-drop

#if running backend locally (in IntelliJ):
#spring.datasource.url=jdbc:postgresql://localhost:5432/postgres

#if running backend in docker container:
spring.datasource.url=jdbc:postgresql://host.docker.internal:5432/postgres

```

## .env 
This must match from application.properties 
```md
POSTGRES_USER=admin
POSTGRES_PASSWORD=admin
```


## backend/Dockerfile

Note: I use JDK17 correto during spring initializer and in IntelliJ
I had bugs running openjdk:17 and various other image specs.
I don't have a good reason why "openjdk:21-ea-1-jdk-slim" works,
but it has worked so far for others as well.


```md
# JDK for docker to use
FROM openjdk:21-ea-1-jdk-slim

# COPY (from local machine) "." to place all on the docker container being created, from "." all
COPY . .

# inside the container use gradle to create a jar file executable to run.
# COPY command is wrong and would copy your local build file, rather than the docker container build file
RUN ./gradlew bootJar

# Run the move command, "mv", from the recently built jar file to a new file called app.jar
# make sure to verify the name "backend-0.0.1.jar" as named in gradle.build file where it says version = '0.0.1' or '0.0.1-SNAPSHOT' just make sure it matches. 
RUN mv build/libs/backend-0.0.1.jar app.jar

# EXPOSE lets devs know what port the container uses. Totally optional:
EXPOSE 8080

# run the app.jar
CMD ["java", "-jar", "app.jar"]
```

## backend/.dockerignore

```md
# Docker doesn't need to copy these when creating the container
Dockerfile
.gradle/
build/
.idea/
```
## frontend/Dockerfile

```md
# use your version of node
FROM node:16 AS build

# within the container, navigate to /app
WORKDIR /app

#copy from local machine package.json to current folder "./"
COPY package*.json ./

# run npm install inside the container
RUN npm install

# Copy files from local machine [ouput all on container] [select all from local machine]
COPY . .

# run build inside container
RUN npm run build

#inside a new container:
# use nginx "engine x"
FROM nginx:alpine
# nginx has tons of functionality, we will use only a
# simple corner of it execute basic HTTP protocols to send information

# Copy from previous container(above) to this one.
# From the container we called "build" to current [path from above] [path on this container]
# double check the name of your project on the first line of package.json,
# this will be the "frontend" of /app/dist/<YOUR PROJECT NAME>
COPY --from=build /app/dist/frontend /usr/share/nginx/html

# EXPOSE indicates primarily to users like you, which port is in use inside Docker
# in my quick test, changing to 81, and and removing entirely (here, not in docker-compose.yml)
# did not stop containers from working correctly.
EXPOSE 80

# Command to execute running the app
CMD ["nginx", "-g", "daemon off;"]
```


## frontend/.dockerignore
```md
# Docker doesn't need to copy these when creating the container
.angular/
.vscode/
dist/
node_modules/
```

# THIS - IS - IT

run: ```docker compose up``` in your terminal

---

---

---

---

---
It should spin the whole application up. You're done.

---

---

---

---

``` I suggest commenting out these for development: ```


``` 
#docker-compose.yml
services:
  database:
    ...
#  backend:
#    ...
#  frontend:
#    ...
```
and switching this for dev:
```md
#application.properties
#local dev:
spring.datasource.url=jdbc:postgresql://localhost:5432/postgres

#inside of docker:
#spring.datasource.url=jdbc:postgresql://host.docker.internal:5432/postgres
```

# Please let me know if I missed something!

# Or if this configuration didn't work for you!








