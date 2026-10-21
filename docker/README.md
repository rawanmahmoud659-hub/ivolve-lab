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
# Part 2: Build Locally

## Objective

Build the Java Calculator application locally using Maven, then create a Docker image using a Java 17 runtime image.

## Step 1: Build the Application Locally

Navigate to the project directory:

```bash
cd ~/ivolve/docker/lab3
```

Build the application using Maven:

```bash
mvn package
```

Check the generated JAR file:

```bash
ls -lh target/*.jar
```

The generated JAR file is:

```text
target/calculator.jar
```

## Step 2: Test the Application Locally

Run the JAR file:

```bash
java -jar target/calculator.jar
```

Enter two numbers and verify the calculator results.

Example input: `8` and `2`.

Expected output:

```text
a = 8.0, b = 2.0
Sum: 10.0
Subtract: 6.0
Multiply: 16.0
Divide: 4.0
```

## Step 3: Create the Dockerfile

Create or edit the Dockerfile:

```bash
nano Dockerfile
```

Dockerfile contents:

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

RUN useradd -m appuser

COPY --chown=appuser:appuser target/calculator.jar app.jar

USER appuser

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

### Explanation

- `FROM`: Uses a Java 17 runtime image.
- `WORKDIR`: Creates and selects `/app` as the working directory.
- `RUN useradd`: Creates a non-root user named `appuser`.
- `COPY`: Copies the generated JAR into the image.
- `USER`: Runs the application as a non-root user.
- `EXPOSE`: Documents the intended port, 8080.
- `CMD`: Starts the Java application.

## Step 4: Build the Docker Image

Build an image named `app2`:

```bash
docker build -t app2 .
```

Check the image and record its size:

```bash
docker images app2
```

**Image size:** Record the actual size shown by Docker.

## Step 5: Run and Verify the Container

Run the application interactively:

```bash
docker run -it --name container2 -p 8080:8080 app2
```

Enter two numbers and verify that the calculator produces the expected results.

This application is a console application, not a web server. Therefore, `curl http://localhost:8080` will not work, even though the Dockerfile contains `EXPOSE 8080`.

## Step 6: Verify the Non-Root User

While the container is running, open another terminal and execute:

```bash
docker exec container2 whoami
```

Expected output:

```text
appuser
```

This confirms that the application container runs as a non-root user.

## Step 7: Remove the Container and Image

Remove the container:

```bash
docker rm -f container2
```

Remove the Docker image:

```bash
docker rmi app2
```

Verify the cleanup:

```bash
docker ps -a --filter name=container2
docker images app2
```

## Conclusion

In Part 2, the Java application was built locally using Maven. Docker then packaged the generated JAR with a Java 17 runtime image instead of including Maven and building the application inside Docker.

This approach generally reduces the final image size by excluding build tools and source code from the runtime image.


```bash
docker ps -a
docker images app1
```
![Docker Lab Screenshot](siz1.png)
![Docker Lab Screenshot](test1.png)
