# Lab 4: Docker Image Optimization and Export/Load

## Objective

The goal of this lab is to:

- Build a Docker image from the original Dockerfile.
- Run and test the application.
- Optimize the Docker image using a multi-stage build.
- Run the application as a non-root user.
- Compare the original and optimized image sizes.
- Export the optimized image to a TAR file.
- Delete the image from the local Docker host.
- Load the image again and verify that it was restored successfully.

---

## 1. Clone the Application

The application was cloned from:

```bash
git clone https://github.com/Ibrahim-Adel15/ImageOptimization-.git
```

Then the project files were placed in the Lab 4 directory.

```bash
cd ~/ivolve/docker/lab4
```

---

## 2. Build the Original Docker Image

The original Dockerfile uses Maven and Java 17 to build and run the application.

Build the image with the name `image1`:

```bash
docker build -t image1 .
```

Chele.

Restored image was verified successfully.ck the image size:

```bash
docker images image1
```
---

## 3. Run the Original Container

Run a container named `container1` from `image1`:

```bash
docker run -d --name container1 -p 8080:8080 image1
```

Check that the container is running:

```bash
docker ps
```

Test the application:

```bash
curl http://localhost:8080
```

The application was successfully started on port `8080`.

Stop and remove the container after testing:

```bash
docker stop container1
docker rm container1
```

---

## 4. Create an Optimized Multi-Stage Dockerfile

The Dockerfile was changed to use a multi-stage build.

### Build Stage

Maven is used only to compile and package the application:

```dockerfile
FROM maven:3.9.15-eclipse-temurin-17-alpine AS build

WORKDIR /app

COPY pom.xml .
COPY src/ ./src

RUN mvn package
```

### Runtime Stage

A smaller Java runtime image is used instead of keeping Maven in the final image:

```dockerfile
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=build --chown=appuser:appgroup /app/target/demo-0.0.1-SNAPSHOT.jar app.jar

USER appuser

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

### Why Multi-Stage Build?

The Maven image contains many build tools that are not required when running the application.

The multi-stage build keeps Maven only in the build stage and copies only the generated JAR file into the final runtime image.

This makes the final image significantly smaller.

---

## 5. Build the Optimized Image

Build the optimized image with the name `image`:

```bash
docker build --no-cache -t image .
```

Check the configured user:

```bash
docker image inspect image --format '{{.Config.User}}'
```

Expected result:

```text
appuser
```

---

## 6. Compare Image Sizes

Check the original image:

```bash
docker images image1
```

Result:

```text
image1 = 1.15 GB
```

Check the optimized image:

```bash
docker images image
```

Result:

```text
image = 302 MB
```

### Comparison

| Image | Size |
|---|---:|
| Original `image1` | 1.15 GB |
| Optimized `image` | 302 MB |

The optimized image is significantly smaller than the original image.

---

## 7. Run the Optimized Container

Run a container named `container2`:

```bash
docker run -d --name container2 -p 8080:8080 image
```

Check that it is running:

```bash
docker ps
```

Test the application:

```bash
curl http://localhost:8080
```

---

## 8. Verify the Container Runs as Non-Root

Check the current user:

```bash
docker exec container2 whoami
```

Expected result:

```text
appuser
```

Check the user ID:

```bash
docker exec container2 id
```

Expected result:

```text
uid=100(appuser) gid=101(appgroup) groups=101(appgroup)
```

This confirms that the application is running as a non-root user instead of `root`.

---

## 9. Stop and Remove the Optimized Container

After testing:

```bash
docker stop container2
docker rm container2
```

Verify that the container was removed:

```bash
docker ps -a
```

---

## 10. Export the Optimized Image

Save the optimized image as a TAR file named `lab-image.tar`:

```bash
docker save -o lab-image.tar image
```

Check the TAR file:

```bash
ls -lh lab-image.tar
```

The generated TAR file was approximately:

```text
85 MB
```

---

## 11. Delete the Local Docker Image

Remove the optimized image from the local Docker host:

```bash
docker rmi image
```

Verify that the image no longer exists:

```bash
docker image inspect image
```

Expected result:

```text
No such image: image:latest
```

---

## 12. Load the Image Again

Restore the image from the TAR file:

```bash
docker load -i lab-image.tar
```

Expected result:

```text
Loaded image: image:latest
```

---

## 13. Verify the Restored Image

Check that the image exists:

```bash
docker images image
```

Check the configured user:

```bash
docker image inspect image --format '{{.Config.User}}'
```

Expected result:

```text
appuser
```

This confirms that the optimized image was successfully restored and still contains the non-root configuration.

---

## Final Results

- Optimized image runs as: **appuser**
- Container name: **container2**
- Export file: **lab-image.tar**
- TAR file size: **approximately 85 MB**
- Image was successfully deleted from the local Docker host.
- Image was successfully loaded again from the TAR file.

Restored image was verified successfully.
