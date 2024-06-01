
# TP 1

## DATABASE

### Dockerfile :
```dockerfile
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

COPY ./sql-scripts/* /docker-entrypoint-initdb.d
```

### Command :
```bash 
docker pull adminer
```
Build the image:
```bash 
docker build -t my-postgres-db
```

Create a network:
```bash
docker network create app-network
```
Run the database container on the network:
```bash
docker run -d --name my-postgres-db --network app-network tpdevops-database
```

Run the adminer container on the network:
```bash
docker run -d --name adminer --network app-network -p 8090:8080 adminer
```



## BACKEND-API

### Dockerfile for multistage build:
```dockerfile
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

First Stage (Build): Uses Maven to compile the project and build the JAR file
Second Stage (Run): Uses a lighter JRE image to run the application. This stage copies the JAR file from the build stage

```bash
javac Main.java
```

Build springboot image:
```bash
docker build -t my-java-app .
```

Run the container:
```bash
docker run --name simple-api-student -p 8080:8080 --network app-network tpdevops-backend-api
```


### Question :
Multistage builds help in creating optimized Docker images by separating the build environment from the runtime environment, reducing the final image size and improving security




## HTTP Server

### Dockerfile :
```dockerfile
FROM httpd:2.4
COPY ./index.html /usr/local/apache2/htdocs/
```

Pull httpd image:
```bash
docker pull httpd
```

Build httpd image:
```bash
docker build -t my-http-server .
```

Run the container:
```bash
docker run -d -p 80:80 --network app-network --name my-http-server tpdevops-frontend-api
```

Copy the configuration to httpd.conf file:
```bash
docker cp httpd.conf my-http-server:/usr/local/apache2/conf/httpd.conf
```

### Question 
We require a reverse proxy to direct client requests to the correct backend service, improve security, facilitate load balancing, and offer a unified entry point for the application.


### Docker Compose

docker-compose.yml file:
```yml
version: '3.7'

services:
    backend:
        container_name: backend-api
        build: ./backend-api/simple-api-student
        networks: 
        - app-network
        depends_on:
        - database

    database: 
        container_name: my_postgrey_app
        build: ./database
        networks: 
        - app-network

    httpd:
        container_name : frontend_app
        build: ./frontend-api
        ports: 
        - "8082:80"
        networks: 
        - app-network
        depends_on:
        - database
        - backend

networks:
    app-network:
```

Run:
```bash
docker compose up
```

### Question 
Docker Compose is crucial because it simplifies the management of multi-container Docker applications, allowing for easy setup and consistency across different environments with a single configuration file. This makes it indispensable for deploying complex applications seamlessly.

## Publication 

## Question 
We've stored our images in a web-based repository to facilitate simple distribution, version control, and deployment among various environments and team members. This approach guarantees uniform and reproducible setups.



# TP 2

At this time my work was on local on my PC so i had to push it on my github repository to test it.

## Test the app

### Command bash :
```bash
mvn clean verify --file simple-api-student/pom.xml
```

### main.yml file :
```yml
name: CI devops 2024
on:
  push:
    branches: 
    - main
    - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2.5.0
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'
      - name: Build and test with Maven
        run: mvn clean verify --file backend-api/simple-api-student/pom.xml

  build-and-push-docker-image:
    needs: test-backend
    runs-on: ubuntu-22.04
    steps:
    - name: Checkout code
      uses: actions/checkout@v2.5.0

    - name: Login to DockerHub
      run: docker login -u ${{ secrets.USERNAME }} -p ${{ secrets.TOKEN }}

    - name: Build image and push backend
      uses: docker/build-push-action@v3
      with:
          context: ./backend-api
          tags: ${{secrets.USERNAME}}/tpdevops-backend-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}

    - name: Build image and push database
      uses: docker/build-push-action@v3
      with:
          context: ./database
          tags: ${{secrets.USERNAME}}/tpdevops-database:latest

    - name: Build image and push httpd
      uses: docker/build-push-action@v3
      with:
          context: ./frontend-api
          tags: ${{secrets.USERNAME}}/tpdevops-frontend-api:latest
```

### Question
Testcontainers are a Java library that uses Docker to create disposable instances for integration tests, ensuring tests are isolated and repeatable. It supports services like databases and web browsers, streamlining testing for applications with external dependencies.


### SonarCloud

On github create a new secret parameters called "SONAR_TOKEN" and give it your personal Sonar Token like : b82a9264c9bf50105f8*****************
