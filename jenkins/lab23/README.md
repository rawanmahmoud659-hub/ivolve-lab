# Lab 23: GitOps with Jenkins and Argo CD

This lab moves Kubernetes deployment from Jenkins to Argo CD. Jenkins updates the container image reference and pushes the manifest to GitHub; Argo CD reconciles the Kubernetes application with the desired state in Git.

## Lab results

The screenshots below capture the deployed application's state. Setup commands and configuration examples provide a guide for reproducing the workflow; adjust environment-specific paths before use.

| Item | Value | Evidence |
| --- | --- | --- |
| Argo CD application | `ivolveapp` | Both screenshots |
| Kubernetes Deployment | `demoapp` | Resource tree |
| Container image | `rawanm21/demoapp:3` | Application summary |
| Health / sync | `Healthy` / `Synced` | Both screenshots |
| Tracked branch / revision | `main` / `413ac13` | Both screenshots |
| Automatic sync | Enabled | Resource tree header |
| Last sync | `Sync OK`; comment `Update image to 3` | Resource tree header |
| Destination | `https://kubernetes.default.svc`, namespace `ivolve` | Application summary |
| Argo CD project | `default` | Application summary |
| Workloads shown | Two running pods, each `1/1`, and three ReplicaSets | Resource tree |

The captured result confirms application health and synchronization. Jenkins build logs and registry push logs are separate verification steps.

## Workflow

```mermaid
flowchart LR
    A[Jenkins: build image] --> B[Push image to Docker Hub]
    B --> C[DeployOnK8s: update manifest]
    C --> D[Commit and push to GitHub main]
    D --> E[Argo CD: detect change and sync]
    E --> F[Kubernetes: roll out demoapp]
```

Jenkins builds and publishes the image before calling the deployment function. Configure the build and registry authentication stages in the pipeline for the application.

`DeployOnK8s` updates the manifest and pushes it to `rawanmahmoud659-hub/ivolve-lab` on branch `main`. Argo CD performs deployment. The function does not run `kubectl apply` or wait for the Kubernetes rollout.

## Prerequisites

- A Kubernetes cluster and an administrator workstation with a working `kubectl` context.
- Jenkins with a configured Shared Library, Git checkout, and Credentials Binding support.
- A Linux Jenkins agent with Git, a shell, and `sed -i` support.
- A pipeline that builds and publishes the requested image tag before updating Git.
- A GitHub username and token with permission to push to the repository's `main` branch, stored as a Jenkins **Username with password** credential named `github-https`. Use the token as the password.
- Argo CD access to the manifest repository. For a private repository, configure repository credentials separately in Argo CD; Jenkins credentials are not automatically shared with it.

## Jenkins Shared Library

Place the following function in `vars/DeployOnK8s.groovy` in the Shared Library repository:

```groovy
def call(String imageName, String newTag, String deploymentFile, String server) {
    withCredentials([
        usernamePassword(credentialsId: 'github-https', usernameVariable: 'USER', passwordVariable: 'TOKEN')
    ]) {
        sh """
            sed -i 's|image: ${imageName}:.*|image: ${imageName}:${newTag}|' ${deploymentFile}

            git config user.name "Jenkins"
            git config user.email "jenkins@localhost"

            git add ${deploymentFile}
            git commit -m "Update image to ${newTag}"

            set +x
            git push https://\$USER:\$TOKEN@github.com/rawanmahmoud659-hub/ivolve-lab.git HEAD:main
        """
    }
}
```

`server` remains in the signature so existing Jenkinsfile calls remain compatible; it is unused. `deploymentFile` must be inside the checked-out repository and within the directory Argo CD watches. The workspace must contain the correct branch history and no unrelated staged changes.

**Example call**, after loading the Shared Library and publishing the image:

```groovy
DeployOnK8s(
    'rawanm21/demoapp',
    '3',
    'Kuberntes/lab23/manifests/deployment.yaml',
    ''
)
```

The example assumes the manifest is located at `Kuberntes/lab23/manifests/deployment.yaml`. Replace this path if the repository uses a different layout, and configure Argo CD to watch the same manifest directory. The image used in the captured deployment is **`rawanm21/demoapp:3`**.

### Implementation notes

- If the tag is unchanged, `git commit` fails because there is nothing to commit. A missing image match can cause the same outcome. Check the manifest diff before diagnosing this as an authentication failure.
- The `sed` expression assumes a simple `image: repository:tag` line and trusted arguments without shell-special characters; it is not a general YAML editor.
- A concurrent update to `main` can cause a rejected push. Refresh the checkout and resolve the change before retrying; do not force-push over others' work.
- `set +x` hides shell tracing of the push, but a token in the URL can still be visible in the agent's process arguments. Use a credential helper or `GIT_ASKPASS` for a hardened implementation.
- If repository pushes trigger Jenkins, configure trigger/path filtering to prevent manifest-only commits from creating a build loop.

## Install and access Argo CD

**Example — fresh lab installation.** Run these Bash commands on the administrator workstation. `stable` is a moving reference; pin a tested release for repeatability. These installation and access steps follow the [official getting-started guide](https://argo-cd.readthedocs.io/en/stable/getting_started/).

```bash
kubectl config current-context
kubectl create namespace argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=180s
kubectl get pods -n argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Keep port forwarding running. Open [Argo CD locally](https://localhost:8080). In another terminal, retrieve the initial password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 --decode
printf '\n'
```

Log in as `admin` and change the initial password. The default installation uses a self-signed certificate. If the optional Argo CD CLI is installed using the [official CLI instructions](https://argo-cd.readthedocs.io/en/stable/cli_installation/), the lab login is:

```bash
argocd login localhost:8080 --username admin --insecure
```

`--insecure` skips certificate verification for this local lab connection.

## Configure the Argo CD application

**Example configuration.** This manifest uses the application name, project, destination, branch, and automatic sync shown in the screenshots. Verify the assumed source directory before applying.

Create the destination namespace if it does not already exist:

```bash
kubectl create namespace ivolve
```

Save this example as `application.yaml` outside the watched workload directory:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ivolveapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rawanmahmoud659-hub/ivolve-lab.git
    targetRevision: main
    path: Kuberntes/lab23/manifests # Verify this assumed directory.
  destination:
    server: https://kubernetes.default.svc
    namespace: ivolve
  syncPolicy:
    automated: {}
```

```bash
kubectl apply -f application.yaml
```

Alternatively, enter those same fields in **Applications → New App** and select automatic sync. The manifest should deploy `demoapp` into `ivolve`; any explicit namespace in the workload manifest must agree with the destination.

`automated: {}` enables automatic synchronization. Automatic pruning and self-healing are separate settings; neither is established by the screenshots, so this example does not enable them. See the [official automated sync documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/).

## Verify the deployment

**Verification commands.** Trigger the Jenkins pipeline with a newly published image tag. Inspect its manifest commit and push, then verify Argo CD and Kubernetes:

```bash
# From the manifest repository checkout:
git log -1 --oneline
git show --stat HEAD

# From the administrator workstation:
kubectl get application ivolveapp -n argocd \
  -o jsonpath='{.status.sync.status}{"\n"}{.status.health.status}{"\n"}{.status.sync.revision}{"\n"}'
kubectl rollout status deployment/demoapp -n ivolve --timeout=120s
kubectl get deployment demoapp -n ivolve \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
kubectl get pods -n ivolve -o wide
```

With the optional authenticated CLI:

```bash
argocd app get ivolveapp --refresh
argocd app wait ivolveapp --sync --health --timeout 180
```

For the captured result, the status is `Synced` and `Healthy`, the revision starts with `413ac13`, and the image is `rawanm21/demoapp:3`. A subsequent successful build should show its own new revision and image tag. The rollout command above is an operator verification step, not part of the revised Jenkins function.

If synchronization fails, inspect Argo CD application conditions, repository access, branch, and source path. For pod failures, use `kubectl describe pod <pod-name> -n ivolve` and inspect events for image-pull or startup errors. These screenshots do not establish a Service, application port, or successful HTTP response, so no application URL is assumed.

## Screenshots

The screenshots document the application summary and resource tree after synchronization.

### Application summary

Shows the `default` project, in-cluster destination, `ivolve` namespace, `Healthy` status, `main` revision `413ac13`, and image `rawanm21/demoapp:3`.

![Argo CD application summary showing the destination, synced revision, health, and container image](screenshots/argocd-application-summary.png)

### Application resource tree

Shows `ivolveapp` managing Deployment `demoapp`, three ReplicaSets, two running `1/1` pods, automatic sync enabled, and a successful sync to `413ac13`.

![Argo CD resource tree showing ivolveapp Healthy and Synced, demoapp, ReplicaSets, and two running pods](screenshots/argocd-resource-tree.png)

## Documentation layout

```text
README.md
screenshots/
├── argocd-application-summary.png
└── argocd-resource-tree.png
```

Keep the `screenshots/` directory beside this README so the images render correctly on GitHub.
