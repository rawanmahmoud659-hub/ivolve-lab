# Lab 21: Automated CI/CD Pipeline with Jenkins, Docker, Trivy, and Kubernetes

## Objective

Implement a Jenkins declarative pipeline that builds a Node.js application, packages it as a Docker image, checks the image for vulnerabilities, publishes it to Docker Hub, and updates an application running in Kubernetes.

**Application repository:** https://github.com/rawanmahmoud659-hub/ivolve-app  
**Docker Hub image:** `rawanm21/ivolve-app`  
**Kubernetes namespace:** `ivolve`  
**Deployment:** `ivolveapp`  
**Docker Hub Jenkins credential ID:** `docker`

## 1. CI/CD Workflow

The Jenkins pipeline executes the following stages in order:

1. **Checkout:** Retrieve the `main` branch from GitHub.
2. **Build App:** Install Node.js dependencies using `npm install` and run tests if a test script exists.
3. **Build Docker Image:** Create `rawanm21/ivolve-app:${BUILD_NUMBER}`.
4. **Scan Docker Image:** Use Trivy to report HIGH and CRITICAL vulnerabilities and save `trivy-report.txt`.
5. **Push Docker Image:** Authenticate with Docker Hub using Jenkins credentials and push the image.
6. **Delete Local Image:** Remove the newly built image from the Jenkins host after pushing.
7. **Update Deployment File:** Replace the application image tag in `deployment.yaml` with the current Jenkins build number.
8. **Deploy to Kubernetes:** Apply the updated manifest and wait for the Deployment rollout.

The pipeline also defines `always`, `success`, and `failure` post-build actions.

## 2. Requirements and Jenkins Configuration

The Jenkins agent needs Git, Node.js, npm, Docker CLI with access to the Docker daemon, Trivy, and `kubectl`. Jenkins runs as a system service in this setup.

### Docker access

The Jenkins service account must have permission to access Docker. On a trusted, dedicated Jenkins host, this can be configured with:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
sudo -u jenkins -H docker info
```

> **Security:** Membership in the `docker` group grants powerful, effectively root-level access to the Docker host. Restrict who can configure or run Jenkins jobs.

### Docker Hub credentials

In **Manage Jenkins → Credentials**, create a **Username with password** credential with ID `docker`. Use the Docker Hub username and an access token as the password. The pipeline injects these credentials only during the push stage.

### Kubernetes access and RBAC

Configure the Jenkins service to use Kubernetes credentials for the `jenkinssa` ServiceAccount in the `ivolve` namespace. The pipeline needs permission to apply Deployments and check rollout status.

Create or update a namespace-scoped Role and RoleBinding using the `create | apply` approach:

```bash
kubectl create role jenkins-role \
  --namespace=ivolve \
  --verb=get,list,watch,create,update,patch \
  --resource=pods,deployments.apps \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create rolebinding jenkins-rolebinding \
  --namespace=ivolve \
  --role=jenkins-role \
  --serviceaccount=ivolve:jenkinssa \
  --dry-run=client -o yaml | kubectl apply -f -
```

Validate the required access:

```bash
kubectl auth can-i patch deployments.apps -n ivolve \
  --as=system:serviceaccount:ivolve:jenkinssa

kubectl auth can-i get pods -n ivolve \
  --as=system:serviceaccount:ivolve:jenkinssa
```

Both commands should return `yes`. The `patch` verb is important because `kubectl apply` may patch an existing Deployment. These RBAC resources grant permissions; Jenkins must separately be configured with valid cluster authentication.

## 3. Jenkinsfile

The following is the pipeline used for this lab. It checks out the application repository, uses the Jenkins build number as the image tag, and deploys to the `ivolve` namespace.

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'rawanm21/ivolve-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDS = 'docker'
        K8S_NAMESPACE = 'ivolve'
        DEPLOYMENT = 'ivolveapp'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/rawanmahmoud659-hub/ivolve-app.git'
            }
        }

        stage('Build App') {
            steps {
                sh '''
                    npm install
                    npm test --if-present
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Scan Docker Image') {
            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --format table \
                      --output trivy-report.txt \
                      ${DOCKER_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDS}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set +x
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'docker logout >/dev/null 2>&1 || true; rm -rf "$DOCKER_CONFIG"' EXIT

                        echo "$DOCKER_PASS" | docker login \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Delete Local Image') {
            steps {
                sh '''
                    docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Update Deployment File') {
            steps {
                sh '''
                    sed -i \
                      "s|image: rawanm21/ivolve-app:.*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|" \
                      deployment.yaml

                    grep 'image:' deployment.yaml
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl apply -f deployment.yaml \
                      -n ${K8S_NAMESPACE}

                    kubectl rollout status \
                      deployment/${DEPLOYMENT} \
                      -n ${K8S_NAMESPACE} \
                      --timeout=180s
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution finished.'
            archiveArtifacts artifacts: 'trivy-report.txt',
                             allowEmptyArchive: true
        }

        success {
            echo 'SUCCESS: Application deployed to Kubernetes!'
        }

        failure {
            echo 'FAILURE: Pipeline failed. Check Jenkins logs.'
        }
    }
}
```

**Scanning behavior:** This Trivy command does **not** set `--exit-code 1`, so reported vulnerabilities do not automatically fail the pipeline. The scan is recorded as a report, not enforced as a deployment-blocking security gate. For a stricter production pipeline, remediate findings and add an appropriate failure threshold.

**Deployment file behavior:** The `sed` command updates the file in the Jenkins workspace. It does not commit the new image tag to GitHub.

## 4. Running the Pipeline

1. Create a Jenkins Pipeline job and configure **Pipeline script from SCM**.
2. Select Git, use the repository URL above, and choose the `main` branch.
3. Set the script path to `Jenkinsfile`.
4. Start the job with **Build Now**.
5. Open **Console Output** to follow the stages, then review the archived Trivy report.

The job log shows Jenkins loading the Jenkinsfile from the application repository and executing each stage successfully.

## 5. Successful Execution — Build #6

The recorded Jenkins run completed all stages:

| Stage | Verified result |
| --- | --- |
| Checkout | Checked out `main` from the application repository |
| Build App | `npm install` completed; `npm test --if-present` executed |
| Build Docker Image | Built and tagged `rawanm21/ivolve-app:6` |
| Scan Docker Image | Trivy ran and wrote `trivy-report.txt` |
| Push Docker Image | Docker Hub login succeeded; image tag `6` was pushed |
| Delete Local Image | `docker rmi rawanm21/ivolve-app:6` completed |
| Update Deployment File | Manifest image updated to `rawanm21/ivolve-app:6` |
| Deploy to Kubernetes | `kubectl apply` configured the Deployment and rollout succeeded |
| Post Actions | Trivy report archived; success message printed |

Selected console output:

```text
Successfully tagged rawanm21/ivolve-app:6
Login Succeeded
6: digest: sha256:99b4a7e8648463f078e9a894c4ea35982bdf4fd18b69d9158db4be11338acf73 size: 2057
Untagged: rawanm21/ivolve-app:6
          image: mysql:8.0
          image: rawanm21/ivolve-app:6
deployment.apps/ivolveapp configured
deployment "ivolveapp" successfully rolled out
Pipeline execution finished.
SUCCESS: Application deployed to Kubernetes!
Finished: SUCCESS
```

Trivy also warned that the detected Alpine 3.21.3 base operating system was no longer supported and that vulnerability detection might be incomplete. The image should be updated to a supported base before production use.

### Docker Hub Push Verification

The following screenshot shows the `rawanm21/ivolve-app` repository on Docker Hub after the Jenkins pipeline pushed the versioned image. Tag `6` is visible alongside earlier tags, confirming that the image was published successfully.

![Docker Hub repository showing the pushed ivolve-app image tags](screenshots/dockerhub.png)

## 6. Kubernetes Deployment Verification

After the pipeline, check the Deployment and Pods:

```bash
kubectl get deployments.apps -n ivolve -o wide
kubectl get pods -n ivolve -o wide
kubectl rollout status deployment/ivolveapp -n ivolve
```

The verified Deployment output was:

```text
NAME        READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                  SELECTOR
ivolveapp   2/2     2            2           12m   ivolveapp    rawanm21/ivolve-app:6   app=ivolveapp
```

This confirms that the Deployment was available with **2 of 2 replicas ready**, using the image pushed by Jenkins build #6.

## 7. Post-Build Actions

- **`always`:** Prints a completion message and archives the Trivy report, including when an earlier stage fails and the report exists.
- **`success`:** Confirms that the pipeline completed and Kubernetes deployment succeeded.
- **`failure`:** Prints an error message directing the user to Jenkins logs for troubleshooting.

## 8. Result

**Lab completed successfully.** Jenkins automated the application build, Docker image creation, vulnerability scanning, Docker Hub publishing, local image cleanup, deployment manifest update, and Kubernetes rollout. Build **#6** published `rawanm21/ivolve-app:6`, and Kubernetes reported `ivolveapp` as **2/2 ready**.

---

**Useful links:** [Application source](https://github.com/rawanmahmoud659-hub/ivolve-app) · [Docker Hub repository](https://hub.docker.com/r/rawanm21/ivolve-app)
