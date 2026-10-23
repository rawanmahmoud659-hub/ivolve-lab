# Lab 8: Containerized Node.js and MySQL Stack Using Docker Compose

## Overview
This lab demonstrates how to containerize a Node.js application and a MySQL database using Docker Compose. The application connects to a MySQL database named `ivolve`, and Docker Compose manages the services, network, and persistent volumes.

## Technologies Used
- Docker
- Docker Compose
- Node.js 18
- MySQL 8.0
- Docker Hub
- Git and GitHub

## Project Structure

```text
lab8/
├── Dockerfile
├── docker-compose.yml
├── db.js
├── package.json
├── server.js
├── frontend/
│   ├── index.html
│   └── assets/
│       └── ivolve-logo.png
└── README.md
```

## 1. Clone the Application

The original application repository:

```bash
git clone https://github.com/Ibrahim-Adel15/iVolveApp.git
```

The application files were organized in the `docker/lab8/` directory of the main project.

## 2. Dockerfile

The Dockerfile uses Node.js 18 Alpine as the base image, installs dependencies, copies the application files, exposes port 3000, and starts the server.

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

## 3. Docker Compose Configuration

The `docker-compose.yml` file defines two services: `app` and `db`.

- **App:** Builds the Node.js image, publishes port `3000`, configures database environment variables, and mounts a persistent logs volume at `/app/logs`.
- **Database:** Uses MySQL 8.0, creates the `ivolve` database, and stores database files in a persistent volume.
- **Network:** Uses a custom bridge network named `ivolve-net` so the application can communicate with MySQL using the hostname `db`.
- **Healthcheck:** Checks MySQL readiness before allowing the application service to start.

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      DB_HOST: db
      DB_USER: root
      DB_PASSWORD: root
      DB_NAME: ivolve
    volumes:
      - app_logs:/app/logs
    depends_on:
      db:
        condition: service_healthy
    networks:
      - ivolve-net

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ivolve
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 5s
      timeout: 5s
      retries: 20
    networks:
      - ivolve-net

volumes:
  app_logs:
  db_data:

networks:
  ivolve-net:
    name: ivolve-net
    driver: bridge
```

## 4. Build and Start the Services

Validate the Compose configuration:

```bash
docker compose config
```

Build the application image and start both services in the background:

```bash
docker compose up -d --build
```

Check the running containers:

```bash
docker compose ps
```

View application logs:

```bash
docker compose logs app
```

View MySQL logs:

```bash
docker compose logs db
```

## 5. Verify Application Functionality

Test the application using curl:

```bash
curl http://localhost:3000
```

The application returned its HTML page successfully and became accessible at:

http://localhost:3000

The application logs confirmed successful MySQL connectivity:

```text
Connected to MySQL and 'ivolve' DB found.
Server started on http://0.0.0.0:3000
```

## 6. Verify Database Connectivity and Data

List the databases:

```bash
docker compose exec db mysql -uroot -proot -e "SHOW DATABASES;"
```

List tables in the `ivolve` database:

```bash
docker compose exec db mysql -uroot -proot -e "USE ivolve; SHOW TABLES;"
```

Retrieve the products:

```bash
docker compose exec db mysql -uroot -proot -e "USE ivolve; SELECT * FROM products;"
```

The application initializes the `products` table and inserts sample products when the table is empty.

## 7. Verify Network and Volumes

Inspect the custom network:

```bash
docker network inspect ivolve-net
```

List the Compose volumes:

```bash
docker volume ls | grep lab8
```

The persistent volumes are:

| Volume | Container path | Purpose |
|---|---|---|
| `lab8_app_logs` | `/app/logs` | Persist application logs |
| `lab8_db_data` | `/var/lib/mysql` | Persist MySQL data |

## 8. Tag and Push the Image to Docker Hub

Log in to Docker Hub:

```bash
docker login
```

List local images:

```bash
docker images | grep ivolve-app
```

Tag the application image for Docker Hub:

```bash
docker tag ivolve-app:v3 rawanm21/ivolve-app:latest
```

Push the image:

```bash
docker push rawanm21/ivolve-app:latest
```

Docker Hub repository:

https://hub.docker.com/r/rawanm21/ivolve-app

**Note:** The image was tagged and prepared for upload. Confirm that the push completed successfully before considering the Docker Hub step finished.

## 9. GitHub Version Control

The Lab 8 files were added to the main project repository:

```bash
cd ~/ivolve
git add docker/lab8
git commit -m "Add Docker Compose Node.js MySQL Lab 8"
git pull --rebase origin main
git push origin main
```

GitHub repository:

https://github.com/rawanmahmoud659-hub/ivolve-lab

## Conclusion

This lab demonstrates how to run a Node.js application and MySQL database using Docker Compose, connect services through a custom network, persist logs and database files using volumes, verify application and database functionality, and publish a Docker image to Docker Hub.
