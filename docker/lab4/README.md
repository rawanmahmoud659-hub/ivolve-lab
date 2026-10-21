
# Lab 4: Docker Image Optimization and Export/Load

## Objective
Optimize a Docker image using a multi-stage build, run the application as a non-root user, and practice saving and loading Docker images.

## 1. Build the Original Image

```bash
docker build -t image1 .
docker images image1
