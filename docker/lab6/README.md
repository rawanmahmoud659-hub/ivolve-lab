# Lab 6: Docker Environment Variables and Resource Management

## Objectives

- Build a Docker image using `python:3.12-slim`.
- Configure environment variables using `-e`, `--env-file`, and Dockerfile `ENV`.
- Run containers with CPU and memory limits.
- Verify application functionality and environment variables.
- Monitor container resource usage using `docker stats`.

## 1. Application and Dockerfile

The application is a Python Flask application that runs on port `5000`.

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

### Explanation

- `FROM python:3.12-slim`: Uses a lightweight Python 3.12 base image.
- `WORKDIR /app`: Sets the working directory inside the container.
- `COPY requirements.txt .`: Copies the dependencies file.
- `RUN pip install --no-cache-dir -r requirements.txt`: Installs the required Python packages.
- `COPY . .`: Copies the application files into the image.
- `EXPOSE 5000`: Documents the port used by the application.
- `CMD ["python", "app.py"]`: Starts the Flask application.

## 2. Build the Docker Image

```bash
docker build -t ivolve1 .
```

**Explanation:**

- `docker build`: Builds a Docker image.
- `-t ivolve1`: Assigns the image name `ivolve1`.
- `.`: Uses the current directory as the build context.

Verify the image:

```bash
docker images
```

## 3. Run Container 1 with Environment Variables

```bash
docker run -d \
  --name container1 \
  -p 5000:5000 \
  -e APP_NAME=iVolveApp1 \
  -e APP_ENV=stg \
  --cpus=1 \
  --memory=256m \
  ivolve1
```

### Explanation

- `-d`: Runs the container in the background.
- `--name container1`: Assigns the container name.
- `-p 5000:5000`: Maps host port 5000 to container port 5000.
- `-e APP_NAME=iVolveApp1`: Sets the application name.
- `-e APP_ENV=stg`: Sets the environment to staging.
- `--cpus=1`: Limits CPU usage to one CPU.
- `--memory=256m`: Limits container memory to 256 MiB.
- `ivolve1`: Specifies the Docker image.

### Verify the Container

```bash
docker ps
docker exec container1 printenv APP_NAME
docker exec container1 printenv APP_ENV
```

Expected environment variables:

```text
iVolveApp1
stg
```

### Test the Application

```bash
curl http://localhost:5000
curl http://localhost:5000/health
```

The application should return its name, environment, and welcome message. The health endpoint should return:

```json
{"status":"healthy"}
```

The application is also accessible through a browser at http://localhost:5000.

## 4. Verify Resource Limits and Monitor Usage

Inspect the configured resource limits:

```bash
docker inspect container1 --format='NanoCPUs={{.HostConfig.NanoCpus}} Memory={{.HostConfig.Memory}}'
```

Expected output:

```text
NanoCPUs=1000000000 Memory=268435456
```

Monitor CPU and memory usage:

```bash
docker stats container1
```

Press `Ctrl+C` to stop monitoring.

**Explanation:** `docker inspect` displays the configured limits, while `docker stats` displays the container's current resource consumption.

## 5. Create an Environment File

Create `env.txt`:

```bash
cat > env.txt <<'EOF'
APP_NAME=iVolveApp2
APP_ENV=prod
EOF
```

View the file:

```bash
cat env.txt
```

### Run Container 2 Using the Environment File

Stop the first container to release host port 5000:

```bash
docker stop container1
```

Run the second container:

```bash
docker run -d \
  --name container2 \
  -p 5000:5000 \
  --env-file env.txt \
  --cpus=1 \
  --memory=256m \
  ivolve1
```

### Verify Environment Variables

```bash
docker exec container2 printenv APP_NAME
docker exec container2 printenv APP_ENV
curl http://localhost:5000
```

Expected environment variables:

```text
iVolveApp2
prod
```

**Explanation:** The `--env-file` option loads environment variables from a file instead of specifying each variable individually with `-e`.

## 6. Set Environment Variables in the Dockerfile

Create a new file named `Dockerfile.dev`:

```dockerfile
FROM ivolve1

ENV APP_NAME=iVolveApp3
ENV APP_ENV=dev
```

### Explanation

- `FROM ivolve1`: Uses the existing application image as the base image.
- `ENV APP_NAME=iVolveApp3`: Defines the default application name.
- `ENV APP_ENV=dev`: Defines the default environment as development.

### Build the New Image

```bash
docker build -f Dockerfile.dev -t ivolve3 .
```

### Remove the Previous Container

```bash
docker rm -f container2
```

The container must be removed before reusing its name.

### Run the New Container

```bash
docker run -d \
  --name container2 \
  -p 5000:5000 \
  --cpus=1 \
  --memory=256m \
  ivolve3
```

### Verify the Environment Variables

```bash
docker exec container2 printenv APP_NAME
docker exec container2 printenv APP_ENV
curl http://localhost:5000
curl http://localhost:5000/health
```

Expected environment variables:

```text
iVolveApp3
dev
```


Monitor the container:

```bash
docker stats container2
```

## 7. Environment Variables Comparison

| Configuration Method | APP_NAME | APP_ENV |
|---|---|---|
| `-e` option | iVolveApp1 | stg |
| `--env-file` option | iVolveApp2 | prod |
| Dockerfile `ENV` | iVolveApp3 | dev |

## 8. Key Learnings

- Docker images package applications and their dependencies.
- Containers are running instances of Docker images.
- Environment variables allow application configuration without modifying the application code.
- The `-e` option sets environment variables when starting a container.
- The `--env-file` option loads environment variables from a file.
- The Dockerfile `ENV` instruction defines default environment variables in an image.
- The `-p` option publishes container ports to the host.
- The `--cpus` and `--memory` options limit container resources.
- `docker inspect` verifies configuration, while `docker stats` monitors actual resource usage.
- Sensitive information should not be hardcoded into Docker images.

## Conclusion

In this lab, I built Docker images for a Python Flask application, configured environment variables using three different methods, limited CPU and memory resources, verified application functionality, and monitored container resource usage.
