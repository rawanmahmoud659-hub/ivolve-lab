# Lab 7: Multi-Network Microservices, Volumes, and Bind Mounts

## Objective
Deploy a microservices application using Docker with:
- Three services: Frontend, Backend, and Notification.
- Two isolated bridge networks.
- Docker volumes for application logs.
- A bind mount for persistent backend data.
- Service-to-service communication using container names.

## 1. Clone the Application

```bash
cd ~/ivolve/docker/lab7
git clone https://github.com/Ibrahim-Adel15/MicroserviceApp.git
cd MicroserviceApp
```

## 2. Create Dockerfiles

Create a separate Dockerfile for each service:

```text
frontend/Dockerfile
backend/Dockerfile
notification/Dockerfile
```

Each Dockerfile defines the base image, working directory, dependencies, application files, exposed port, and startup command.

## 3. Build Docker Images

```bash
docker build -t frontend:lab7 ./frontend
docker build -t backend:lab7 ./backend
docker build -t notification:lab7 ./notification
```

Verify the images:

```bash
docker images
```

## 4. Create Docker Networks

Create two isolated bridge networks with different subnets:

```bash
docker network create \
  --driver bridge \
  --subnet 172.28.0.0/16 \
  frontend-backend

docker network create \
  --driver bridge \
  --subnet 172.29.0.0/16 \
  backend-notification
```

Verify the networks:

```bash
docker network ls
docker network inspect frontend-backend
docker network inspect backend-notification
```

## 5. Create Docker Volumes

Create volumes to persist backend and notification logs:

```bash
docker volume create backend-logs
docker volume create notification-logs
```

Verify the volumes:

```bash
docker volume ls
```

## 6. Create the Backend Data Directory

Create a host directory to persist backend application data:

```bash
mkdir -p ~/ivolve/docker/lab7/backend-data
```

This directory will be mounted inside the backend container at `/app/data`.

## 7. Run the Notification Container

```bash
docker run -d \
  --name notification \
  --network backend-notification \
  --mount source=notification-logs,target=/app/logs \
  notification:lab7
```

The notification service is connected to the `backend-notification` network.

## 8. Run the Backend Container

Start the backend on the `frontend-backend` network, configure the notification URL, and mount the persistent data directory and log volume:

```bash
docker run -d \
  --name backend \
  --network frontend-backend \
  -e NOTIFICATION_URL=http://notification:5001/notify \
  --mount type=bind,source="$HOME/ivolve/docker/lab7/backend-data",target=/app/data \
  --mount source=backend-logs,target=/app/logs \
  backend:lab7
```

Connect the backend to the second network:

```bash
docker network connect backend-notification backend
```

The backend is now connected to both networks.

## 9. Run the Frontend Container

```bash
docker run -d \
  --name frontend \
  --network frontend-backend \
  -e BACKEND_URL=http://backend:5000 \
  -p 8080:3000 \
  frontend:lab7
```

Open the application in a browser:

```text
http://localhost:8080
```

The host port `8080` is mapped to the frontend container port `3000`.

## 10. Verify the Deployment

Check running containers:

```bash
docker ps
```

Check container logs:

```bash
docker logs frontend
docker logs backend
docker logs notification
```

Inspect network connectivity:

```bash
docker network inspect frontend-backend
docker network inspect backend-notification
```

Verify persistent backend data:

```bash
cat ~/ivolve/docker/lab7/backend-data/messages.json
```

Check application log files:

```bash
docker exec backend ls -l /app/logs
docker exec notification ls -l /app/logs
```

## 11. Test the Application

1. Open `http://localhost:8080`.
2. Create a test message using the application interface.
3. Verify that the message can be retrieved.
4. Check the backend data directory to confirm persistence.
5. Review the backend and notification logs.

## Expected Results

- Three Docker images are built successfully.
- Three containers run: frontend, backend, and notification.
- The frontend and backend share the `frontend-backend` network.
- The backend and notification share the `backend-notification` network.
- Backend and notification logs are stored in Docker volumes.
- Backend application data is persisted using a bind mount.
- The frontend is accessible on port `8080`.
- Messages can be created and retrieved successfully.

**Note:** Container ports, environment variable names, notification endpoints, and mount paths must match the actual application code. The commands above assume the services use ports `3000`, `5000`, and `5001`, respectively.
