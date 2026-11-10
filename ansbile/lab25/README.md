# Lab 25: Automating Docker, Kubectl, and Jenkins Installation with Ansible

## Objective

Create a single Ansible playbook to automate the installation of **Docker Engine**, **kubectl**, and **Jenkins** on an Ubuntu managed node. Configure Docker and Jenkins to start automatically and verify all installations.

## Prerequisites

- Ansible installed on the control node
- Ubuntu managed node with SSH connectivity and a user with sudo privileges
- Internet access on the managed node to download packages and repository keys
- An inventory file named `inventory`

## Project Structure

```text
lab25/
├── inventory
└── install_tools.yaml
```

## Step 1: Configure Inventory

Create `inventory` on the control node:

```ini
[managed]
ansible-managed ansible_host=172.31.10.25 ansible_user=ubuntu
```

Replace the example IP with your managed node's address. If required, add `ansible_ssh_private_key_file=/path/to/key.pem`.

Test SSH connectivity:

```bash
ansible -i inventory managed -m ping
```

Expected response: `pong`.

## Step 2: Create the Playbook

Create `install_tools.yaml`:

```yaml
---
- name: Install Docker, Kubectl and Jenkins
  hosts: managed
  become: true

  tasks:
    - name: Install prerequisites
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - python3-apt
          - openjdk-21-jre
        state: present
        update_cache: true

    - name: Create APT keyrings directory
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    # Docker
    - name: Add Docker GPG key
      ansible.builtin.shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg |
          gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        chmod a+r /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg

    - name: Add Docker repository
      ansible.builtin.apt_repository:
        repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        filename: docker
        state: present

    - name: Install Docker
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
        update_cache: true

    - name: Start and enable Docker
      ansible.builtin.systemd:
        name: docker
        state: started
        enabled: true

    - name: Add ubuntu user to Docker group
      ansible.builtin.user:
        name: ubuntu
        groups: docker
        append: true

    # Kubectl
    - name: Add Kubernetes GPG key
      ansible.builtin.shell: |
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key |
          gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg
      args:
        creates: /etc/apt/keyrings/kubernetes.gpg

    - name: Add Kubernetes repository
      ansible.builtin.apt_repository:
        repo: "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /"
        filename: kubernetes
        state: present

    - name: Install Kubectl
      ansible.builtin.apt:
        name: kubectl
        state: present
        update_cache: true

    # Jenkins
    - name: Add Jenkins GPG key
      ansible.builtin.get_url:
        url: https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
        dest: /etc/apt/keyrings/jenkins-keyring.asc
        mode: '0644'

    - name: Add Jenkins repository
      ansible.builtin.apt_repository:
        repo: "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/"
        filename: jenkins
        state: present

    - name: Install Jenkins
      ansible.builtin.apt:
        name: jenkins
        state: present
        update_cache: true

    - name: Start and enable Jenkins
      ansible.builtin.systemd:
        name: jenkins
        state: started
        enabled: true
```

### How the Playbook Works

- **`hosts: managed`** targets the hosts in the `[managed]` inventory group.
- **`become: true`** uses privilege escalation (normally sudo) for system administration tasks.
- **APT repository and GPG key tasks** configure trusted package sources for Docker, Kubernetes, and Jenkins.
- **`creates:`** skips shell commands when the key file already exists, avoiding repeated downloads. It does not verify that the existing key is current.
- **`state: present`** installs missing packages without reinstalling existing ones unnecessarily.
- **`systemd`** starts Docker and Jenkins and enables them at boot.
- **`append: true`** adds the `ubuntu` user to the Docker group without removing existing supplementary groups.

## Step 3: Run the Playbook

Check syntax:

```bash
ansible-playbook -i inventory install_tools.yaml --syntax-check
```

Run the playbook:

```bash
ansible-playbook -i inventory install_tools.yaml
```

If sudo requires a password, append `--ask-become-pass`.

## Step 4: Verify Installation

Run these commands from the Ansible control node:

```bash
ansible -i inventory managed -a 'docker --version'
ansible -i inventory managed -a 'kubectl version --client'
ansible -i inventory managed -a 'java -version'
ansible -i inventory managed -b -a 'systemctl is-active docker'
ansible -i inventory managed -b -a 'systemctl is-active jenkins'
```

Expected results:

| Component | Expected result |
| --- | --- |
| Docker | Docker version displayed |
| kubectl | Client version displayed |
| Java | Java 21 version displayed |
| Docker service | `active` |
| Jenkins service | `active` |

### Observed Results on the Managed Node

The installation run showed `changed` for the Docker, Kubernetes, and Jenkins installation/repository tasks. Docker and Jenkins service tasks completed without errors in the visible output. The final play recap was not shown, so no recap totals are assumed.

Additional checks performed directly on the managed node:

```bash
docker ps
kubectl version
systemctl status jenkins.service
```

- **Docker:** `docker ps` returned the container table without an error. The table contained no running containers at the time of the check, confirming that Docker was accessible.
- **kubectl:** The installed client reported **v1.35.10**, with **Kustomize v5.7.1**.
- **Kubernetes API authorization:** Running `kubectl version` also attempted to contact the Kubernetes API server and returned **Forbidden / Authentication required**, identifying the user as `anonymous` without `hudson.model.Hudson.Read` permission. This indicates an authorization/configuration issue with the endpoint being contacted, **not a failed kubectl installation**. Use `kubectl version --client` to verify the client without contacting a server, and check the active kubeconfig/context before using kubectl against a cluster.
- **Jenkins:** `systemctl status jenkins.service` showed **active (running)** and **enabled**. Jenkins was running with Java and configured to listen on port **8080**.

## Step 5: Access Jenkins

Visit `http://<MANAGED_NODE_PUBLIC_IP>:8080` after allowing TCP port 8080 from **your IP only** in the instance firewall/security group.

Retrieve the initial administrator password securely:

```bash
ansible -i inventory managed -b -a 'cat /var/lib/jenkins/secrets/initialAdminPassword'
```

Do not commit this password to Git.

## Idempotency

Ansible aims for the desired system state. On a subsequent run, tasks should generally return `ok` rather than `changed` when packages and services are already configured. Some repository or key operations may still require maintenance when keys or versions change.

## Notes

- This example assumes **Ubuntu on amd64** and uses the Kubernetes **v1.35** package repository. Match the kubectl version to your cluster compatibility requirements.
- Membership in the `docker` group grants powerful root-equivalent access; only add trusted users.
- The playbook installs the kubectl CLI, **not** a Kubernetes cluster.
- For production use, consider key rotation, repository version pinning, and separate Ansible roles (introduced in Lab 26).

## Conclusion

Lab 25 demonstrates how a single Ansible playbook can automate DevOps tool installation, configure package repositories, start required services, and verify Docker, kubectl, and Jenkins on a managed Ubuntu node.
