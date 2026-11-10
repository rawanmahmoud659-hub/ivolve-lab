# Lab 26: Structured Configuration Management with Ansible Roles

## Objective

Refactor the installation of **Docker**, **kubectl**, and **Jenkins** into three reusable Ansible roles and execute them from a single playbook. This lab builds on Lab 25, where all installation tasks were kept in one playbook.

## Environment

- **Control node:** `ansible-control` (Ubuntu, Ansible installed)
- **Managed node:** `ansible-managed` (Ubuntu, reachable over SSH)
- **Privilege escalation:** `become: true` (sudo)
- **Inventory file:** `inventory`

> The configuration below assumes an Ubuntu **amd64** managed node and an SSH user named `ubuntu`. Adjust these values for your environment. Do not install Docker on an existing Kubernetes node without reviewing its container-runtime dependencies.

## Project Structure

```text
.
├── inventory
├── install_tools.yaml
└── roles/
    ├── docker/
    │   └── tasks/
    │       └── main.yml
    ├── kubectl/
    │   └── tasks/
    │       └── main.yml
    └── jenkins/
        └── tasks/
            └── main.yml
```

`ansible-galaxy role init` also creates optional role directories such as `handlers/`, `defaults/`, `vars/`, `files/`, `templates/`, `meta/`, and `tests/`. The installation logic in this lab is inside each role's `tasks/main.yml`.

## 1. Create the Roles

On the control node, from your project directory:

```bash
mkdir -p roles
ansible-galaxy role init roles/docker
ansible-galaxy role init roles/kubectl
ansible-galaxy role init roles/jenkins
```

Each role is independent and can be reused in other playbooks.

## 2. Configure `inventory`

```ini
[managed]
ansible-managed ansible_host=<MANAGED_NODE_IP> ansible_user=ubuntu
```

Replace `<MANAGED_NODE_IP>` with the actual reachable IP address. The group name `managed` must match `hosts: managed` in the playbook. Configure SSH key authentication as appropriate for your environment.

Test connectivity:

```bash
ansible -i inventory managed -m ping
```

## 3. Docker Role — `roles/docker/tasks/main.yml`

```yaml
---
- name: Install Docker prerequisites
  apt:
    name:
      - ca-certificates
      - curl
      - gnupg
      - python3-apt
    state: present
    update_cache: true

- name: Create APT keyrings directory
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Add Docker GPG key
  shell: |
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg |
      gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    chmod a+r /etc/apt/keyrings/docker.gpg
  args:
    creates: /etc/apt/keyrings/docker.gpg

- name: Add Docker repository
  apt_repository:
    repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    filename: docker
    state: present

- name: Install Docker
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present
    update_cache: true

- name: Start and enable Docker
  systemd:
    name: docker
    state: started
    enabled: true

- name: Add ubuntu user to Docker group
  user:
    name: ubuntu
    groups: docker
    append: true
```

**Explanation:** Installs prerequisites, creates the APT keyring directory, imports Docker's repository signing key, configures the official Docker APT repository, installs Docker Engine and related plugins, and starts/enables the Docker service. The final task grants the `ubuntu` user access to the Docker socket; **Docker group membership effectively grants root-level access**, so only use it for trusted users. `creates` skips the key-import command if the output file already exists; it does not check whether the key is current.

## 4. kubectl Role — `roles/kubectl/tasks/main.yml`

```yaml
---
- name: Install kubectl prerequisites
  apt:
    name:
      - ca-certificates
      - curl
      - gnupg
      - python3-apt
    state: present
    update_cache: true

- name: Create APT keyrings directory
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Add Kubernetes GPG key
  shell: |
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key |
      gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg
  args:
    creates: /etc/apt/keyrings/kubernetes.gpg

- name: Add Kubernetes repository
  apt_repository:
    repo: "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /"
    filename: kubernetes
    state: present

- name: Install kubectl
  apt:
    name: kubectl
    state: present
    update_cache: true
```

**Explanation:** Adds the Kubernetes package-signing key and the Kubernetes v1.35 repository, then installs `kubectl`. This installs **only the Kubernetes CLI**, not a Kubernetes cluster. Match the kubectl minor version to your cluster's supported version skew.

## 5. Jenkins Role — `roles/jenkins/tasks/main.yml`

```yaml
---
- name: Install Java and Jenkins prerequisites
  apt:
    name:
      - openjdk-21-jre
      - ca-certificates
      - curl
      - python3-apt
    state: present
    update_cache: true

- name: Create APT keyrings directory
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Add Jenkins GPG key
  get_url:
    url: https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
    dest: /etc/apt/keyrings/jenkins-keyring.asc
    mode: '0644'

- name: Add Jenkins repository
  apt_repository:
    repo: "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/"
    filename: jenkins
    state: present

- name: Install Jenkins
  apt:
    name: jenkins
    state: present
    update_cache: true

- name: Start and enable Jenkins
  systemd:
    name: jenkins
    state: started
    enabled: true
```

**Explanation:** Installs Java 21 and prerequisites, adds the Jenkins signing key and stable APT repository, installs Jenkins, and starts/enables its systemd service. Jenkins typically listens on port **8080**; restrict network access appropriately.

## 6. Main Playbook — `install_tools.yaml`

```yaml
---
- name: Install Docker, Kubectl and Jenkins
  hosts: managed
  become: true

  roles:
    - docker
    - kubectl
    - jenkins
```

**Explanation:** `hosts: managed` targets the inventory group, `become: true` executes privileged tasks using sudo, and `roles:` loads each role's `tasks/main.yml` in the listed order. The `---` marker denotes the start of the YAML document.

## 7. Run the Playbook

Check syntax:

```bash
ansible-playbook -i inventory install_tools.yaml --syntax-check
```

Execute:

```bash
ansible-playbook -i inventory install_tools.yaml
```

If sudo prompts for a password, use `--ask-become-pass`.

## 8. Verify the Installation

Run these commands from the **control node**:

```bash
ansible -i inventory managed -a 'docker --version'
ansible -i inventory managed -a 'kubectl version --client'
ansible -i inventory managed -a 'java -version'
ansible -i inventory managed -b -a 'systemctl is-active docker'
ansible -i inventory managed -b -a 'systemctl is-active jenkins'
```

Expected results: Docker, kubectl, and Java report their installed versions; both `systemctl is-active` commands return `active`.

## 9. Results and Idempotency

A successful **repeat run** of the playbook produced this recap:

```text
PLAY RECAP
ansible-managed : ok=14 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

- **`ok=14`** — 14 tasks completed successfully.
- **`changed=0`** — no changes were needed on this run.
- **`unreachable=0`** — Ansible could connect to the managed node.
- **`failed=0`** — no tasks failed.

This demonstrates **idempotency**: once the desired configuration is present, rerunning the playbook does not unnecessarily modify it. Note that a file-presence guard such as `creates` does not validate or refresh an existing signing key.

## Conclusion

Lab 26 demonstrates how Ansible Roles make configuration management more modular, reusable, and maintainable. Docker, kubectl, and Jenkins are installed through three separate roles coordinated by one playbook, with successful repeat execution confirming the configured state.
