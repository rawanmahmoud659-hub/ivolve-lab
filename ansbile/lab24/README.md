# Lab 24: Initial Ansible Configuration and Ad-Hoc Execution

## Objective
Configure Ansible on an AWS EC2 control node, establish SSH access to a managed EC2 instance, create an inventory, and install Docker remotely using an Ansible ad-hoc command.

## Architecture

| Node | AWS EC2 instance | Purpose |
| --- | --- | --- |
| Control node | `ansible-control` | Runs Ansible and stores the inventory and SSH key |
| Managed node | `ansible-slave` (inventory alias: `ansible-managed`) | Receives configuration and runs Docker |

Both nodes run Ubuntu. Use the managed instance's **private IPv4 address** for communication inside the same VPC (or otherwise connected networks). The managed node's security group must allow inbound TCP/22 from the control node's security group.

> **Security:** Never commit private keys, `.pem` files, or secrets to GitHub. Replace example IP addresses with your actual values.

## 1. Install Ansible on the control node

Connect to `ansible-control` using EC2 Instance Connect or SSH, then run:

```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv python3-full -y

sudo apt install pipx -y
pipx ensurepath
pipx install --include-deps ansible
```

If `which ansible` points to `/usr/bin/ansible` instead of the pipx installation, prioritize the pipx executable:

```bash
export PATH="$HOME/.local/bin:$PATH"
hash -r
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
which ansible
ansible --version
```

**Observed verification:** Ansible was available from `/home/ubuntu/.local/bin/ansible`, reporting **ansible-core 2.21.6**. The pipx package was **ansible 14.6.0**.

## 2. Generate an SSH key on the control node

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/ansible_key
chmod 600 ~/.ssh/ansible_key
ls -l ~/.ssh/
```

This creates:

- `~/.ssh/ansible_key`: **private** key; keep it on the control node.
- `~/.ssh/ansible_key.pub`: public key; copy it to the managed node.

If the files already exist, do not overwrite them without checking whether they are in use. For a noninteractive lab you can leave the passphrase empty; for stronger security use a passphrase and an SSH agent.

## 3. Transfer the public key to the managed node

First, confirm initial SSH access to the managed node with its original AWS key pair or another authorized authentication method. `ssh-copy-id` cannot bootstrap access without an already working login.

With the original AWS private key available **securely on the control node**, for example at `~/.ssh/aws-key.pem`, use:

```bash
chmod 600 ~/.ssh/aws-key.pem
ssh-copy-id -i ~/.ssh/ansible_key.pub \
  -o IdentityFile="$HOME/.ssh/aws-key.pem" \
  ubuntu@<MANAGED_PRIVATE_IP>
```

Then verify the new key:

```bash
ssh -i ~/.ssh/ansible_key ubuntu@<MANAGED_PRIVATE_IP>
exit
```

If `ssh-copy-id` says `No such file`, confirm the `.pub` file exists with `ls -l ~/.ssh/`. If it says `Permission denied (publickey)`, fix the initial SSH authentication before retrying.

## 4. Create the inventory

On the control node:

```bash
mkdir -p ~/ansible-labs/lab24
cd ~/ansible-labs/lab24
nano inventory
```

Add the following, replacing `<MANAGED_PRIVATE_IP>` with the real private IP:

```ini
[managed]
ansible-managed ansible_host=<MANAGED_PRIVATE_IP> ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/ansible_key ansible_python_interpreter=/usr/bin/python3
```

The explicit Python path avoids interpreter auto-discovery warnings when `/usr/bin/python3` is the intended interpreter.

Check the inventory and connectivity:

```bash
ansible-inventory -i inventory --graph
ansible managed -i inventory -m ping
```

A successful connection returns `SUCCESS` and `"ping": "pong"`. Ansible's `ping` module tests SSH/Python connectivity, not ICMP.

## 5. Install Docker with an Ansible ad-hoc command

Run on the **control node**:

```bash
ansible managed -i inventory -b -m apt -a "name=docker.io state=present"
```

| Argument | Explanation |
| --- | --- |
| `managed` | Target inventory group |
| `-i inventory` | Inventory file |
| `-b` | Become root via sudo |
| `-m apt` | Use Ansible's APT module |
| `name=docker.io` | Ubuntu Docker package |
| `state=present` | Install if missing; do nothing if already installed |

**Observed result:** The ad-hoc task returned `ansible-managed | CHANGED`, with `changed: true` and `stderr: ""`, indicating Docker was installed. The output also showed a Python interpreter auto-discovery warning for `/usr/bin/python3.14`; this is a warning, not a failed task.

For future installations, `update_cache=yes` can be included to refresh package metadata:

```bash
ansible managed -i inventory -b -m apt -a "name=docker.io state=present update_cache=yes"
```

## 6. Verify Docker on the managed node

From the control node:

```bash
ansible managed -i inventory -m command -a "docker --version"
ansible managed -i inventory -b -m command -a "systemctl is-active docker"
ansible managed -i inventory -b -m command -a "docker version"
```

Or connect to `ansible-slave` and run:

```bash
docker --version
sudo docker version
sudo systemctl status docker --no-pager
```

**Observed output:** The managed node reported Docker Client **29.1.3**. Running `docker version` as the unprivileged `ubuntu` user printed:

```text
permission denied while trying to connect to the docker API at unix:///var/run/docker.sock
```

This means the Docker CLI is installed, but the user cannot access the daemon socket. It does **not** by itself mean the Docker installation failed. For verification, use `sudo docker version`.

Optional: if non-root Docker access is required for the lab, add the user to the `docker` group and reconnect:

```bash
sudo usermod -aG docker ubuntu
# Log out and reconnect, then:
docker version
```



