# Automate User Creation on Linux Server using Ansible

## Project Overview

This project demonstrates how to automate user creation and management on Linux servers using Ansible. Managing user accounts manually across multiple servers can be tedious and error-prone. This solution leverages Ansible playbooks to streamline user provisioning, group assignments, and SSH key management.

## Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Setup Instructions](#setup-instructions)
- [Usage](#usage)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
 
## Architecture

```
Control Machine (Ansible Controller)
    ├── Ansible Installed
    ├── SSH Keys configured
    ├── Inventory File
    └── Playbooks
          └── Target Servers
              ├── User1 (created with sudo group)
              ├── User2 (created with docker group)
              └── Additional users
```

## Prerequisites

### Hardware/Infrastructure Requirements
- **Control Machine**: 1 Linux server (Ubuntu 24.04 recommended)
- **Target Server(s)**: 1+ Linux servers for user management
- **Network**: SSH access (port 22) between control and target machines

### Software Requirements
- Ansible 2.9+ installed on control machine
- Python 3.6+ on all servers
- SSH client and server configured
- Sudo privileges on all servers

### Access Requirements
- AWS EC2 key pair (`.pem` file) for initial server access
- SSH key-based authentication configured
- Internet connectivity for package installation

## Project Structure

```
ansible-user-automation/
├── README.md
├── inventory.ini              # Inventory file with target servers
├── create_users.yml           # Main playbook for user creation
├── backup.yml                 # Backup playbook (optional)
├── restore.yml                # Restore playbook (optional)
└── .ssh/
    ├── id_rsa                 # Control machine private key
    ├── id_rsa.pub             # Control machine public key
    ├── wordpress.pem          # AWS EC2 key pair
    ├── user1_key              # SSH key for user1 (optional)
    ├── user1_key.pub
    ├── user2_key              # SSH key for user2 (optional)
    └── user2_key.pub
```

## Setup Instructions

### Step 1: Install Ansible on Control Machine

```bash
# Update package repository
sudo apt update

# Install Ansible
sudo apt install ansible -y

# Verify installation
ansible --version
```

### Step 2: Configure SSH Key-Based Authentication

#### Generate SSH Key Pair (if not already created)

```bash
ssh-keygen -t rsa -b 4096
# Press Enter to accept default location (~/.ssh/id_rsa)
# Press Enter for no passphrase (or set one for security)
```

#### Copy Public Key to Target Servers

**Method 1: Using wordpress.pem key**

```bash
# Copy your control machine's public key to target
ssh -i ~/.ssh/wordpress.pem ubuntu@<target-server-ip> \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
   echo '$(cat ~/.ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys && \
   chmod 600 ~/.ssh/authorized_keys"
```

**Method 2: Using ssh-copy-id (if password auth is enabled)**

```bash
ssh-copy-id ubuntu@<target-server-ip>
ssh-copy-id -f "-o IdentityFile /Users/celeb/Downloads" ubuntu@13.222.130.176
```

#### Test Passwordless Connection

```bash
ssh ubuntu@<target-server-ip>
# Should connect without password prompt
```

### Step 3: Create Ansible Inventory File

```bash
# Create project directory
mkdir -p ~/ansible-user-automation
cd ~/ansible-user-automation

# Create inventory file
nano inventory.ini
```

**Add the following content:**

```ini
[linux_servers]
target ansible_host=<target-server-ip> ansible_user=ubuntu

[web_servers]
webserver ansible_host=<webserver-ip> ansible_user=ubuntu
```

**Example:**

```ini
[linux_servers]
target ansible_host=13.222.130.176 ansible_user=ubuntu

[web_servers]
webserver ansible_host=54.90.136.152 ansible_user=ubuntu
```

### Step 4: Test Ansible Connectivity

```bash
# Test connection to all servers
ansible -i inventory.ini all -m ping

# Expected output:
# target | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

### Step 5: Create User Automation Playbook

```bash
nano create_users.yml
```

**Add the following playbook:**

```yaml
---
- name: Automate user creation
  hosts: linux_servers
  become: yes
  tasks:
    - name: Create users with groups and passwords
      user:
        name: "{{ item.username }}"
        state: present
        shell: /bin/bash
        create_home: yes
        groups: "{{ item.groups }}"
        password: "{{ item.password | password_hash('sha512') }}"
      with_items:
        - { username: "user1", groups: "sudo", password: "ChangeMe123!" }
        - { username: "user2", groups: "docker", password: "ChangeMe456!" }
        - { username: "devops", groups: "sudo,docker", password: "DevOps2024!" }
```

**Advanced Version with SSH Keys:**

```yaml
---
- name: Automate user creation
  hosts: linux_servers
  become: yes
  tasks:
    - name: Create users with groups
      user:
        name: "{{ item.username }}"
        state: present
        shell: /bin/bash
        create_home: yes
        groups: "{{ item.groups }}"
      with_items:
        - { username: "user1", groups: "sudo" }
        - { username: "user2", groups: "docker" }
    
    - name: Add SSH keys for the users
      authorized_key:
        user: "{{ item.username }}"
        state: present
        key: "{{ lookup('file', item.ssh_key) }}"
      with_items:
        - { username: "user1", ssh_key: "~/.ssh/user1_key.pub" }
        - { username: "user2", ssh_key: "~/.ssh/user2_key.pub" }
```

## Usage

### Run the User Creation Playbook

```bash
# Execute the playbook
ansible-playbook -i inventory.ini create_users.yml

# Expected output:
# PLAY [Automate user creation] ****************************
# 
# TASK [Gathering Facts] ***********************************
# ok: [target]
# 
# TASK [Create users with groups and passwords] ************
# changed: [target] => (item={'username': 'user1', 'groups': 'sudo', 'password': 'ChangeMe123!'})
# changed: [target] => (item={'username': 'user2', 'groups': 'docker', 'password': 'ChangeMe456!'})
# changed: [target] => (item={'username': 'devops', 'groups': 'sudo,docker', 'password': 'DevOps2024!'})
# 
# PLAY RECAP ***********************************************
# target      : ok=2    changed=1    unreachable=0    failed=0
```

### Run Ad-Hoc Commands

#### Check Uptime

```bash
ansible -i inventory.ini linux_servers -m command -a "uptime"
```

#### Check Disk Usage

```bash
ansible -i inventory.ini linux_servers -m shell -a "df -h"
```

#### Verify User Creation

```bash
ansible -i inventory.ini linux_servers -m shell -a "cat /etc/passwd | grep -E 'user1|user2'" --become
```

## Verification

### Method 1: Using Ansible Ad-Hoc Commands

```bash
# Check if users exist
ansible -i inventory.ini target -m command -a "id user1" --become
ansible -i inventory.ini target -m command -a "id user2" --become

# List home directories
ansible -i inventory.ini target -m command -a "ls -la /home" --become

# Check user groups
ansible -i inventory.ini target -m shell -a "groups user1" --become
ansible -i inventory.ini target -m shell -a "groups user2" --become
```

### Method 2: SSH into Target Server

```bash
# Connect to target server
ssh ubuntu@<target-server-ip>

# Check users in /etc/passwd
cat /etc/passwd | grep -E "user1|user2|devops"

# Check home directories
ls -la /home/

# Verify group memberships
groups user1
groups user2
groups devops

# Try switching to a created user
sudo su - user1
# Enter password: ChangeMe123!

# Exit back to ubuntu user
exit

# Exit server
exit
```

### Method 3: Test SSH Access with Keys (if configured)

```bash
# Connect as user1 using SSH key
ssh -i ~/.ssh/user1_key user1@<target-server-ip>

# Connect as user2 using SSH key
ssh -i ~/.ssh/user2_key user2@<target-server-ip>
```

## Troubleshooting

### Issue 1: Permission Denied (publickey)

**Problem:** Cannot connect to target server

```bash
ubuntu@54.83.101.160: Permission denied (publickey).
```

**Solution:**

```bash
# Add your public key to the target server
ssh -i ~/.ssh/wordpress.pem ubuntu@<target-ip> \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
   echo '$(cat ~/.ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys && \
   chmod 600 ~/.ssh/authorized_keys"
```

### Issue 2: Host Pattern Not Matched

**Problem:**

```
[WARNING]: Could not match supplied host pattern, ignoring: web_servers
```

**Solution:** Update your playbook's `hosts:` line to match your inventory groups:

```yaml
# Change from:
hosts: web_servers

# To:
hosts: linux_servers
```

### Issue 3: Module Parameter Error

**Problem:**

```
"msg": "Unsupported parameters for (ansible.legacy.copy) module: format"
```

**Solution:** Use the correct module - `archive` instead of `copy` for compression:

```yaml
# Incorrect:
- name: Copy files
  copy:
    src: /path
    dest: /backup
    format: gz  # ❌ copy doesn't support format

# Correct:
- name: Create backup archive
  archive:
    path: /home/ubuntu
    dest: /backup/backup.tar.gz
    format: gz  # ✅ archive supports format
```

### Issue 4: Permission Denied Creating Directory

**Problem:**

```
"msg": "There was an issue creating /backup as requested: [Errno 13] Permission denied"
```

**Solution:** Add `become: yes` to your playbook:

```yaml
---
- name: Backup files
  hosts: webserver
  become: yes  # ← Add this line
  tasks:
    - name: Create backup directory
      file:
        path: /backup
        state: directory
```

### Issue 5: AWS EC2 IP Addresses Keep Changing

**Problem:** Public IPs change when stopping/starting instances

**Solution:** Use AWS Elastic IP addresses:

```bash
# In AWS Console:
# 1. Navigate to EC2 → Elastic IPs
# 2. Allocate new Elastic IP
# 3. Associate with your instance

# Update inventory.ini with the new static IP
```

## Best Practices

### Security

1. **Use Strong Passwords**: Always use strong, unique passwords for users
   ```yaml
   password: "Use_C0mplex_P@ssw0rds!"
   ```

2. **SSH Key Authentication**: Prefer SSH keys over passwords
   ```bash
   # Generate unique keys for each user
   ssh-keygen -t rsa -f ~/.ssh/user1_key -C "user1@company"
   ```

3. **Limit Sudo Access**: Only grant sudo to users who need it
   ```yaml
   groups: "docker"  # Don't add to sudo unnecessarily
   ```

4. **Protect Private Keys**: Set proper permissions on SSH keys
   ```bash
   chmod 400 ~/.ssh/wordpress.pem
   chmod 400 ~/.ssh/user1_key
   ```

![ans](images/1.png)
![ans](images/2.png)
![ans](images/3.png)
![ans](images/4.png)
![ans](images/5.png)
![ans](images/ansible-1.png)
![ans](images/ansible-2.png)
![ans](images/ansible-3.png)
![ans](images/ansible-4.png)
![ans](images/ansible-5.png)
![ans](images/ansible-6.png)
![ans](images/ansible-7.png)
![ans](images/ansible-8.png)
![ans](images/ansible-9.png)
![ans](images/ansible-10.png)
![ans](images/ansible-11.png)
![ans](images/ansible-12.png)
![ans](images/ansible-13.png)
![ans](images/ansible-14.png)
![ans](images/ansible-15.png)
![ans](images/ansible-16.png)
![ans](images/ansible-17.png)
![ans](images/ansible-18.png)
![ans](images/ansible-19.png)
![ans](images/ansible-20.png)
![ans](images/ansible-nginx-configured.png)