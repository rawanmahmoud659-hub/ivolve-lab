# Lab 3: Creating a Docker Image for a Java Application

## Part 1: Build Inside Docker

### 1. Clone the Repository
```bash
git clone https://github.com/Ibrahim-Adel15/CalculatorMavenApp.git
cd CalculatorMavenApp
```

### 2. Create the Dockerfile
Create a Dockerfile using Maven with Java 17, create a non-root user, copy the source code, and build the application.

```bash
nano Dockerfile
```

### 3. Build the Docker Image
Build the application inside Docker and create an image named `app1`.

```bash
docker build -t app1 .
```

### 4. Check the Image Size
Display the image size and record it.

```bash
docker images app1
```

### 5. Run the Container
Run the application in a container named `container1`.

```bash
docker run --name container1 app1
```

### 6. Verify the Application
Check the container logs.

```bash
docker logs container1
```

Verify that the application runs as a non-root user (while the container is running).

```bash
docker exec container1 whoami
```

Expected output:
```text
appuser
```

### 7. Remove the Container and Image
Delete the container and the Docker image after verification.

```bash
docker rm -f container1
docker rmi app1
```

### 8. Verify Cleanup
Confirm that the container and image have been removed.

```bash
docker ps -a
docker images app1
```
![Docker Lab Screenshot](siz1.png)
![Docker Lab Screenshot](test1.png)
