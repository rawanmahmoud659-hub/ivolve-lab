# Lab 22: CI/CD Pipeline Implementation with Jenkins Agents and Shared Libraries

## Objective

Implement a Jenkins CI/CD pipeline on a **Jenkins agent (slave)** using three reusable Jenkins Shared Library steps:

1. **BuildApp** — run unit tests and build the application.
2. **BuildImage** — build a Docker image, scan it with Trivy, push it to Docker Hub, and delete the local image.
3. **DeployOnK8s** — update the image in a Kubernetes Deployment manifest, apply it, and wait for rollout.

**Source application:** https://github.com/Ibrahim-Adel15/DemoApp.git  
**Example Docker Hub image:** `rawanm21/demoapp`  
**Kubernetes namespace:** `ivolve`  
**Deployment:** `demoapp`  
**Kubernetes ServiceAccount:** `jenkinssa`

> This README provides a complete reference implementation based on the lab workflow. Adjust the application build/test commands to the actual build system in the cloned DemoApp repository. Do not claim a successful pipeline run until its console output and deployment have been verified.

## 1. Architecture

```text
DemoApp GitHub repository
          |
          v
Jenkins controller -> Jenkins agent (Docker + Trivy + kubectl)
                         |
                         +--> BuildApp: test + package
                         |
                         +--> BuildImage: build + scan + push + local cleanup
                         |
                         +--> DeployOnK8s: edit YAML + kubectl apply + rollout
                                                        |
                                                        v
                                               Kubernetes namespace ivolve
```

The Jenkins agent uses Docker to build images. Kubernetes continues using **containerd** as its container runtime. If Docker is installed on a Kubernetes node, package changes and runtime/network effects must be checked carefully.

## 2. Clone the application

On the Jenkins agent, or through the pipeline checkout stage:

```bash
git clone https://github.com/Ibrahim-Adel15/DemoApp.git
cd DemoApp
ls -la
find . -maxdepth 2 \( -name Dockerfile -o -name pom.xml -o -name build.gradle -o -name package.json \)
```

This repository supplies the application and Dockerfile. Check whether it uses Maven, Gradle, npm, or another build system before selecting test/build commands. The examples below assume **Maven**; change them if DemoApp uses a different build tool.

## 3. Configure the Jenkins agent

In **Manage Jenkins → Nodes → New Node**, create a permanent agent, choose a label such as `docker-agent`, and configure its remote working directory (for example `/home/ubuntu/jenkins-agent`). Set up the agent launch method (SSH or inbound agent) and confirm the node is **Online**.

Install the required tools **on the agent**, not just on the controller: Git, the application build tool/JDK, Docker Engine/CLI, Trivy, and kubectl.

### 3.1 Docker and existing containerd

If the agent is also a Kubernetes node, simulate installation first:

```bash
sudo apt-get update
sudo apt-get -s install docker.io
```

**Stop** if the simulation removes or replaces `containerd`, `runc`, `kubelet`, or other cluster-critical packages. Installing `docker-ce` from Docker's repository may propose removing Ubuntu's existing `containerd`/`runc`; do not proceed in that case.

If the `docker.io` simulation is safe:

```bash
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo docker version
```

If the agent is a dedicated non-Kubernetes machine, install Docker using the package method appropriate to that host.

### 3.2 Allow the agent user to access Docker

For an SSH-launched agent running as `ubuntu`:

```bash
sudo groupadd -f docker
sudo usermod -aG docker ubuntu
id ubuntu
ls -l /var/run/docker.sock
```

**Disconnect and reconnect the Jenkins agent** after changing group membership; the running agent process otherwise retains its old groups. Check from a Jenkins job:

```groovy
sh 'whoami; id; docker ps'
```

Do **not** use `chmod 666 /var/run/docker.sock`. Docker group access is effectively root-level access; only grant it to trusted agent users/jobs.

### 3.3 Install Trivy

```bash
sudo apt-get update
sudo apt-get install -y wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main' | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
trivy --version
```

For the 2 GB RAM EC2 instance, avoid `/tmp` tmpfs for Trivy temporary files and retain the database cache across builds:

```bash
mkdir -p /home/ubuntu/trivy-tmp /home/ubuntu/.cache/trivy
chmod 700 /home/ubuntu/trivy-tmp
findmnt -T /home/ubuntu/trivy-tmp
```

The pipeline explicitly sets `TMPDIR` and `TRIVY_CACHE_DIR` because Jenkins `sh` steps do **not** reliably source interactive `~/.bashrc` settings. Ensure these paths are disk-backed and writable by the agent user.

### 3.4 Verify tools on the agent

```bash
git --version
java -version
mvn -version       # if Maven is used
kubectl version --client
docker version
trivy --version
```

## 4. Create Kubernetes RBAC for Jenkins

Run these commands with a Kubernetes administrator identity. The ServiceAccount is **namespaced**; ClusterRoles and ClusterRoleBindings are **cluster-scoped**.

```bash
kubectl create serviceaccount jenkinssa -n ivolve --dry-run=client -o yaml | kubectl apply -f -
```

For this lab, Jenkins needs to read pods and create/update/patch/read Deployments. The following **namespaced Role** is the least-privilege option for deployment in `ivolve`:

```bash
kubectl apply -f - <<'YAML'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-deployer
  namespace: ivolve
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
YAML

kubectl create rolebinding jenkins-deployer-binding \
  --role=jenkins-deployer \
  --serviceaccount=ivolve:jenkinssa \
  -n ivolve --dry-run=client -o yaml | kubectl apply -f -
```

If your lab specifically requires **ClusterRole + ClusterRoleBinding**, you can use these instead, but be aware that they grant the listed deployment permissions **in all namespaces**:

```bash
kubectl create clusterrole jenkins-pod-reader \
  --verb=get,list,watch,create,update,patch \
  --resource=pods,deployments.apps \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create clusterrolebinding jenkins-pod-reader-binding \
  --clusterrole=jenkins-pod-reader \
  --serviceaccount=ivolve:jenkinssa \
  --dry-run=client -o yaml | kubectl apply -f -
```

You need **either** the namespaced Role/RoleBinding or the cluster-wide ClusterRole/ClusterRoleBinding for this example, not both. If you require `kubectl get pods -A`, grant cluster-wide read access separately.

Verify:

```bash
kubectl auth can-i patch deployments -n ivolve --as=system:serviceaccount:ivolve:jenkinssa
kubectl auth can-i get deployments -n ivolve --as=system:serviceaccount:ivolve:jenkinssa
kubectl auth can-i list pods -n ivolve --as=system:serviceaccount:ivolve:jenkinssa
```

### 4.1 ServiceAccount token and CA

Prefer short-lived tokens with renewal for production. For this training lab, a manually managed long-lived ServiceAccount token Secret can be used:

```bash
kubectl apply -f - <<'YAML'
apiVersion: v1
kind: Secret
metadata:
  name: jenkinssa-token
  namespace: ivolve
  annotations:
    kubernetes.io/service-account.name: jenkinssa
type: kubernetes.io/service-account-token
YAML
```

Check the token exists **without printing it to logs**:

```bash
kubectl get secret jenkinssa-token -n ivolve -o jsonpath='{.data.token}' | base64 -d | wc -c
```

On the control-plane node, the CA certificate is normally `/etc/kubernetes/pki/ca.crt`. Obtain the token securely and upload the CA certificate to Jenkins Credentials. Never commit a token or kubeconfig to Git. **Rotate any token previously exposed in a console log or chat.**

## 5. Configure Jenkins credentials and Shared Library

In **Manage Jenkins → Credentials → System → Global credentials**, add:

| ID | Kind | Purpose |
|---|---|---|
| `dockerhub-creds` | Username with password | Docker Hub username and access token |
| `k8s-token` | Secret text | Rotated `jenkinssa` token |
| `k8s-ca` | Secret file | Kubernetes API CA certificate (`ca.crt`) |

In **Manage Jenkins → System → Global Trusted Pipeline Libraries** (or the Shared Libraries configuration available in your Jenkins version), register the Git repository containing your library, set its name to `ivolve-shared-lib`, and configure the default branch/version. Use a trusted repository controlled by administrators because library code can access credentials.

Example repository layout:

```text
jenkins-sharedlib/
└── vars/
    ├── BuildApp.groovy
    ├── BuildImage.groovy
    └── DeployOnK8s.groovy
```

The names are deliberately capitalized to match the three required Lab 22 steps. Each file exposes a `call(...)` function invoked from the Jenkinsfile.

## 6. Shared Library #1 — BuildApp

**File:** `vars/BuildApp.groovy`

```groovy
def call() {
    stage('Unit Test') {
        sh 'mvn -B test'
    }
    stage('Build Application') {
        sh 'mvn -B -DskipTests package'
    }
}
```

**Explanation:** `mvn test` executes unit tests. `mvn package` produces the application artifact. Tests are skipped during packaging only because they already ran in the preceding stage. **If DemoApp uses npm or Gradle, replace these commands with the actual project commands.**

## 7. Shared Library #2 — BuildImage

**File:** `vars/BuildImage.groovy`

```groovy
def call(String imageName, String imageTag) {
    withEnv([
        "IMAGE_NAME=${imageName}",
        "IMAGE_TAG=${imageTag}",
        'TMPDIR=/home/ubuntu/trivy-tmp',
        'TRIVY_CACHE_DIR=/home/ubuntu/.cache/trivy'
    ]) {
        try {
            stage('Build Docker Image') {
                sh '''
                    set -eu
                    docker build -t "$IMAGE_NAME:$IMAGE_TAG" .
                '''
            }
            stage('Scan Image with Trivy') {
                sh '''
                    set -eu
                    mkdir -p "$TMPDIR" "$TRIVY_CACHE_DIR"
                    trivy image --severity HIGH,CRITICAL --exit-code 0 \
                      "$IMAGE_NAME:$IMAGE_TAG"
                '''
            }
            stage('Push Docker Image') {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        set +x
                        printf '%s' "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push "$IMAGE_NAME:$IMAGE_TAG"
                        docker logout
                    '''
                }
            }
        } finally {
            stage('Delete Local Docker Image') {
                sh '''
                    docker image rm "$IMAGE_NAME:$IMAGE_TAG" || true
                '''
            }
        }
    }
}
```

**Explanation:** The image is built from the Dockerfile, scanned for HIGH/CRITICAL vulnerabilities, pushed to Docker Hub, and removed locally to conserve disk space. `--exit-code 0` **reports vulnerabilities without failing the build**; change it to `--exit-code 1` if your security policy should block vulnerable images. The `finally` block attempts local cleanup even if a preceding stage fails. Jenkins credentials are expanded by the shell, not by Groovy string interpolation.

## 8. Shared Library #3 — DeployOnK8s

**File:** `vars/DeployOnK8s.groovy`

```groovy
def call(String imageName, String newTag, String deploymentFile) {
    withCredentials([
        string(credentialsId: 'k8s-token', variable: 'TOKEN'),
        file(credentialsId: 'k8s-ca', variable: 'K8S_CA')
    ]) {
        withEnv([
            "IMAGE_NAME=${imageName}",
            "IMAGE_TAG=${newTag}",
            "DEPLOYMENT_FILE=${deploymentFile}"
        ]) {
            sh '''
                set -eu
                set +x

                sed -i "s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" "$DEPLOYMENT_FILE"

                DEPLOY_NAME=$(awk '/^metadata:/{f=1;next} f&&/^  name:/{print $2;exit}' "$DEPLOYMENT_FILE")
                NAMESPACE=$(awk '/^metadata:/{f=1;next} f&&/^  namespace:/{print $2;exit}' "$DEPLOYMENT_FILE")
                test -n "$DEPLOY_NAME"
                test -n "$NAMESPACE"

                kubectl --server=https://172.31.7.13:6443 \
                  --token="$TOKEN" --certificate-authority="$K8S_CA" \
                  apply --dry-run=client -f "$DEPLOYMENT_FILE"

                kubectl --server=https://172.31.7.13:6443 \
                  --token="$TOKEN" --certificate-authority="$K8S_CA" \
                  apply -f "$DEPLOYMENT_FILE"

                kubectl --server=https://172.31.7.13:6443 \
                  --token="$TOKEN" --certificate-authority="$K8S_CA" \
                  rollout status "deployment/$DEPLOY_NAME" -n "$NAMESPACE" --timeout=120s
            '''
        }
    }
}
```

**Explanation:** `sed` changes the image tag in the manifest, `kubectl apply` updates the cluster, and `kubectl rollout status` waits for the Deployment to become ready. Jenkins injects the ServiceAccount token and CA securely through credentials. The example uses the lab's control-plane API address; the Jenkins agent must be able to reach it. Avoid `--insecure-skip-tls-verify`. The simple `awk` extraction assumes the manifest formatting below.

**Optional improvement:** Store a restricted kubeconfig as a Jenkins secret file and set `KUBECONFIG` for the deployment step, avoiding repeated `--server`, `--token`, and CA flags.

## 9. Correct Kubernetes deployment manifest

**File:** `deployment.yaml` in the application repository:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapp
  namespace: ivolve
  labels:
    app: demoapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demoapp
  template:
    metadata:
      labels:
        app: demoapp
    spec:
      containers:
        - name: demoapp
          image: rawanm21/demoapp:1
          resources: {}
```

**Fixes from the original YAML:** Use `namespace: ivolve` instead of `metadata: ivolve`; indent `name` and `resources` under the same `-` list item as `image`. The incorrect indentation caused `yaml: line 20: did not find expected '-' indicator`.

## 10. Jenkinsfile — invoke all three libraries

**File:** `Jenkinsfile` in the application/pipeline repository:

```groovy
@Library('ivolve-shared-lib') _

pipeline {
    agent { label 'docker-agent' }

    environment {
        IMAGE_NAME = 'rawanm21/demoapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout DemoApp') {
            steps {
                git branch: 'main', url: 'https://github.com/Ibrahim-Adel15/DemoApp.git'
            }
        }
        stage('BuildApp') {
            steps {
                script { BuildApp() }
            }
        }
        stage('BuildImage') {
            steps {
                script { BuildImage(env.IMAGE_NAME, env.IMAGE_TAG) }
            }
        }
        stage('DeployOnK8s') {
            steps {
                script { DeployOnK8s(env.IMAGE_NAME, env.IMAGE_TAG, 'deployment.yaml') }
            }
        }
    }
    post {
        success { echo 'CI/CD pipeline completed successfully.' }
        failure { echo 'CI/CD pipeline failed. Check the console output.' }
    }
}
```

**Important:** Confirm the repository's actual default branch and the location of `deployment.yaml`. If the deployment manifest is stored in a **different repository** from DemoApp, check out that repository or copy the manifest into the workspace before calling `DeployOnK8s`. Ensure `BuildApp` matches the actual application build tool and the Dockerfile's expected build artifacts.

## 11. Run and verify the pipeline

1. Register the Shared Library in Jenkins.
2. Configure the agent label and credentials.
3. Create or update a Pipeline job using the Jenkinsfile.
4. Click **Build Now** and inspect **Console Output**.
5. Verify that all three Shared Library stages completed and that the image was pushed.

Run with an administrator kubeconfig:

```bash
kubectl get deployment demoapp -n ivolve
kubectl get pods -n ivolve -l app=demoapp
kubectl rollout status deployment/demoapp -n ivolve
kubectl get deployment demoapp -n ivolve -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

The last command should show `rawanm21/demoapp:<Jenkins-build-number>`. Check Docker Hub for the matching image tag and use `docker images` on the agent to confirm local cleanup.

## 12. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `permission denied ... docker.sock` | Jenkins agent process lacks Docker socket group access | Add the **agent user** to `docker`, then reconnect/restart the agent; verify `id` inside the pipeline |
| Trivy downloads DB again or fills `/tmp` | Pipeline does not inherit `.bashrc`, or cache differs between builds | Set `TMPDIR` and `TRIVY_CACHE_DIR` in pipeline/Shared Library and use persistent disk-backed directories |
| `A secret was passed to sh using Groovy String interpolation` | Secret interpolated inside `sh """..."""` | Use `withCredentials` plus `sh '''...$TOKEN...'''` |
| `yaml: line 20: did not find expected '-' indicator` | Container list indentation incorrect | Use corrected `deployment.yaml` above |
| `BadRequest` connecting to API | HTTP used for TLS API endpoint | Use `https://...:6443` and the CA certificate |
| `Forbidden` on Deployments | ServiceAccount RBAC lacks required verb | Verify `kubectl auth can-i` and grant only required verbs |
| `Authentication required ... hudson.model.Hudson.Read` | kubectl is contacting Jenkins instead of the Kubernetes API | Correct the Kubernetes API server URL/kubeconfig |
| Rollout times out | Image pull, scheduling, readiness, or resource issue | Check `kubectl describe pods -n ivolve` and events |

## 13. Security and resource considerations

- Never store Kubernetes tokens or Docker Hub tokens in GitHub or print them in build logs.
- Rotate previously exposed ServiceAccount tokens. Long-lived Secrets are suitable only when their lifecycle is managed.
- Prefer namespaced Kubernetes RBAC over cluster-wide write permissions.
- Prefer TLS verification using the Kubernetes CA; do not use insecure TLS flags in the final pipeline.
- On a 2 GB EC2 instance, limit concurrent Docker builds and Trivy scans; monitor RAM, CPU, and disk.
- Docker group membership gives powerful host privileges; use trusted agents and jobs.
- If Jenkins agents are ephemeral, use a persistent cache volume or expect Trivy to download its DB again.

## Lab 22 requirements checklist

- [x] Document cloning `Ibrahim-Adel15/DemoApp`.
- [x] Define `BuildApp` (unit test and build).
- [x] Define `BuildImage` (build, Trivy scan, push, local deletion).
- [x] Define `DeployOnK8s` (edit manifest, apply, rollout).
- [x] Provide a Jenkinsfile that invokes all three Shared Libraries.
- [x] Document Jenkins agent configuration and Docker access.
- [x] Document Kubernetes RBAC, Jenkins credentials, and verification.

> The checklist confirms **documentation coverage**, not that every stage has been executed successfully in your environment.
