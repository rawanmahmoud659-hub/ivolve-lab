# Lab 27: Securing Sensitive Data with Ansible Vault

**Course:** Red Hat Enterprise Linux Automation with Ansible (RH294)  
**Objective:** Use an Ansible role to install and configure MySQL on a managed node, while protecting the database password with Ansible Vault.

## Lab requirements

- Install MySQL on the managed node.
- Create a database named `ivolve`.
- Create the MySQL user `ivolveuser` and grant it all privileges on `ivolve`.
- Encrypt the database password with Ansible Vault.
- Verify that the new user can connect and list databases.

> This implementation uses Ubuntu hosts and `apt`. On RHEL, package and service names may differ.

## 1. Lab environment

- **Control node:** `ansible-control`
- **Managed node:** `ansible-managed`
- **Managed node IP:** `172.31.7.179` (example from this lab; replace for another environment)
- **SSH user:** `ubuntu`

The final project structure is:

```text
/home/ubuntu/
├── inventory
├── mysql-playbook.yaml
├── secrets.yaml                 # Encrypted with Ansible Vault
└── roles/
    ├── docker/
    ├── jenkins/
    ├── kubectl/
    └── mysql/
        ├── tasks/
        │   └── main.yml
        ├── defaults/
        ├── handlers/
        ├── meta/
        └── vars/
```

## 2. Configure the inventory

Create or edit the inventory on the control node:

```bash
nano ~/inventory
```

Contents used in this lab:

```ini
[managed]
ansible-managed ansible_host=172.31.7.179 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/id_rsa
```

- `[managed]` is the group targeted by the playbook.
- `ansible-managed` is the inventory hostname.
- `ansible_host` specifies the actual destination IP.
- `ansible_user` specifies the remote SSH account.
- `ansible_ssh_private_key_file` selects the SSH private key.

Verify host discovery and SSH access:

```bash
ansible-inventory -i inventory --graph
ansible managed -i inventory -m ping
```

`--graph` displays inventory groups and hosts. The `ping` module tests Ansible connectivity (it is not an ICMP ping).

## 3. Create the MySQL role

From `/home/ubuntu`, run:

```bash
ansible-galaxy init mysql
```

This generates the standard Ansible role directories, including `tasks/main.yml`. In this lab, the role was initially created at `~/mysql`, outside the role search directory. Move it into `~/roles`:

```bash
mv ~/mysql ~/roles/
ls ~/roles/
```

Expected role names include `docker`, `jenkins`, `kubectl`, and `mysql`.

**Important:** `~/main.yml` is not automatically loaded as the role's task file. If the MySQL tasks were written there, copy them into the correct location:

```bash
cp ~/main.yml ~/roles/mysql/tasks/main.yml
```

Only do this if `~/main.yml` actually contains the MySQL task list.

## 4. Encrypt the database password

Create the encrypted variables file:

```bash
ansible-vault create ~/secrets.yaml
```

Ansible asks for a new Vault password and opens an editor. Enter a variable such as:

```yaml
---
db_password: "REPLACE_WITH_A_STRONG_PASSWORD"
```

Save and exit the editor. **The Vault password unlocks the file; `db_password` is the separate MySQL account password.** Do not commit the Vault password or plaintext credentials to Git.

Useful Vault commands:

```bash
ansible-vault view ~/secrets.yaml
ansible-vault edit ~/secrets.yaml
```

- `view` decrypts the file for display after prompting for the Vault password.
- `edit` decrypts it in an editor and encrypts it again on save.

In the actual lab, the encrypted variable was named **`db_password`**. Every task referencing it must use `{{ db_password }}`—not `{{ database_password }}`.

## 5. Install the MySQL Ansible collection

```bash
ansible-galaxy collection install community.mysql
```

This installs the collection providing the MySQL database and user modules used in the original lab. The installed version displayed deprecation warnings recommending `ansible.mysql.mysql_user` and `ansible.mysql.mysql_db` for future use; these warnings did not cause the successful run to fail.

## 6. Write the role tasks

Edit the role's task file:

```bash
nano ~/roles/mysql/tasks/main.yml
```

The following is a **recommended cleaned-up version** of the successful lab workflow. It uses `db_password` (the actual Vault variable), creates the database before granting permissions, and hides password-bearing task output.

```yaml
---
- name: Install MySQL and Python connector
  ansible.builtin.apt:
    name:
      - mysql-server
      - python3-pymysql
    state: present
    update_cache: true

- name: Start and enable MySQL service
  ansible.builtin.service:
    name: mysql
    state: started
    enabled: true

- name: Create MySQL database 'ivolve'
  community.mysql.mysql_db:
    name: ivolve
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock

- name: Create user and grant privileges on ivolve
  community.mysql.mysql_user:
    name: ivolveuser
    host: localhost
    password: "{{ db_password }}"
    priv: "ivolve.*:ALL"
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock
  no_log: true
```

**What each task does:**

1. **Install MySQL:** `apt` installs `mysql-server` and `python3-pymysql`; `state: present` ensures they exist. `update_cache: true` refreshes apt package metadata.
2. **Start MySQL:** `service` starts MySQL immediately and `enabled: true` starts it automatically on boot.
3. **Create database:** `mysql_db` creates `ivolve` if it does not already exist. The local Unix socket allows administrative access under the typical Ubuntu MySQL setup.
4. **Create user and grant access:** `mysql_user` sets `ivolveuser`'s password from Vault and grants `ALL` privileges on `ivolve.*` (not all databases). `no_log: true` prevents password-bearing module arguments/results from appearing in Ansible logs.

**Original lab note:** The first successful run used `python3-mysqldb`, a separate user-creation task with `plugin_auth_string`, and a separate privilege-grant task. That produced a connector deprecation warning. The cleaned-up example above is an improvement, not a claim that this exact revised file was executed in the captured run.

## 7. Create the playbook

```bash
nano ~/mysql-playbook.yaml
```

```yaml
---
- name: Configure MySQL DB
  hosts: managed
  become: true
  vars_files:
    - secrets.yaml
  roles:
    - mysql
```

- `hosts: managed` selects the `[managed]` inventory group.
- `become: true` uses privilege escalation (`sudo`) to install packages and administer MySQL.
- `vars_files` loads the encrypted `secrets.yaml` file.
- `roles: - mysql` executes the tasks from `~/roles/mysql/tasks/main.yml`.

## 8. Preview and run

First, check that Ansible finds the role tasks:

```bash
ansible-playbook -i inventory mysql-playbook.yaml --ask-vault-pass --list-tasks
```

- `-i inventory` specifies the inventory file.
- `--ask-vault-pass` prompts for the Vault decryption password.
- `--list-tasks` shows tasks without executing them.

Then execute:

```bash
ansible-playbook -i inventory mysql-playbook.yaml --ask-vault-pass
```

If the remote Ubuntu user requires a sudo password, add `-K` to prompt for it.

### Actual successful execution

The lab run completed with:

```text
PLAY RECAP
ansible-managed : ok=6 changed=3 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

The executed tasks were: Gather Facts, Install MySQL, Start MySQL service, Create MySQL user, Create MySQL database `ivolve`, and Grant privileges. `changed=3` means three tasks modified the remote host. The zero failure/unreachable counts confirm the playbook completed successfully; they do **not** replace a database login test.

## 9. Verify MySQL on the managed node

SSH into the managed host from the control node:

```bash
ssh -i ~/.ssh/id_rsa ubuntu@172.31.7.179
```

Check the MySQL service:

```bash
sudo systemctl is-active mysql
```

Expected result: `active`.

Log in using the newly created MySQL user:

```bash
mysql -u ivolveuser -p
```

Enter the **database user password** stored in Vault, not the Vault unlock password. Inside MySQL, run:

```sql
SHOW DATABASES;
USE ivolve;
SHOW GRANTS FOR CURRENT_USER();
EXIT;
```

- `SHOW DATABASES;` confirms `ivolve` is visible.
- `USE ivolve;` verifies that the account can select the database.
- `SHOW GRANTS FOR CURRENT_USER();` verifies privileges on `ivolve.*`.
- `EXIT;` leaves the MySQL client.

**Verification status:** The provided terminal output confirms successful playbook execution. The manual login and database listing were requested but their results were not shown, so this final verification still needs to be performed.

## 10. Troubleshooting encountered

| Error or symptom | Cause | Fix |
|---|---|---|
| `Unable to read source file .../secrets.yaml` | Vault file was not yet created or the path was wrong | Use `ansible-vault create ~/secrets.yaml`, or specify the correct existing path |
| `cp: cannot stat '../inventory'` | The command looked in `/home`, while `inventory` was already in `/home/ubuntu` | No copy needed; use `-i ~/inventory` |
| `Could not match supplied host pattern: slaves` | The playbook targeted `slaves`, but the inventory group was `[managed]` | Set `hosts: managed` |
| `--list-tasks` showed no tasks | MySQL tasks were in `~/main.yml`, not the role's `tasks/main.yml` | Move the role to `~/roles/mysql` and copy the task list into its `tasks/main.yml` |
| `database_password is undefined` | Vault defined `db_password`, but the task referenced `database_password` | Use `{{ db_password }}` consistently |
| `Decryption failed` | Incorrect Vault password or mismatched encrypted file | Verify with `ansible-vault view ~/secrets.yaml` and the original Vault password |
| Python interpreter warning | Ansible auto-detected `/usr/bin/python3.14` | Informational; optionally configure a stable interpreter path after verifying the managed host |
| MySQL module/connector deprecation warnings | Older module aliases or `MySQLdb` connector | Prefer maintained `ansible.mysql` modules and PyMySQL for future updates |

## 11. Security and cleanup

- Replace the short test password used during the lab with a strong, unique database password. Update the Vault file using `ansible-vault edit ~/secrets.yaml`, then rerun the playbook to apply the new password.
- Never paste real credentials into README files or commit plaintext secrets.
- Keep `secrets.yaml` encrypted, and restrict access to the Vault password.
- Grant the database user only the privileges required; this lab deliberately limits `ALL` to `ivolve.*`.
- Use `no_log: true` for tasks that handle secrets.
- Rerun the playbook to check idempotency: once the desired state is reached, tasks should generally report `ok` rather than `changed`.

## Outcome

The Ansible control node successfully executed a MySQL role against the managed node, creating the `ivolve` database and `ivolveuser` account with database-scoped privileges while reading its password from an encrypted Vault file. The captured playbook result was **6 OK, 3 changed, 0 failed**. Complete the manual MySQL login and `SHOW DATABASES;` test to close the lab's final validation requirement.
