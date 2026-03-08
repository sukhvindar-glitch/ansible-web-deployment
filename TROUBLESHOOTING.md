# Troubleshooting Guide - Ansible Web Deployment Project

## Overview
This document covers all errors encountered during the deployment of a web application using Ansible on AWS EC2 instances, along with the troubleshooting steps performed to resolve them.

---

## 1. AWS EC2 Instance Connect - SSH Connection Error

### Error
```
Error establishing SSH connection to your instance. Try again later.
```

### Cause
The Security Group inbound rules were missing the EC2 Instance Connect IP range for us-east-1.

### Fix
Added the following inbound rule to the Security Group:
- **Type:** SSH
- **Port:** 22
- **Source:** `18.206.107.24/29` (AWS EC2 Instance Connect IP range for us-east-1)

---

## 2. Private Key File Incorrectly Moved

### Error
```
mv ~/Control-node.pem ~/.ssh/~
```
The `.pem` file was accidentally moved and renamed to `~` instead of being placed inside `~/.ssh/`.

### Cause
Incorrect use of `mv` command — trailing `~` was interpreted as a filename instead of the home directory shortcut.

### Fix
Renamed the misplaced file from inside `~/.ssh/`:
```bash
mv '~' Control-node.pem
```
Single quotes were used to treat `~` as a literal filename rather than expanding it to the home directory path.

---

## 3. Ansible Ping - Inventory File Not Found

### Error
```
[WARNING]: Unable to parse /home/ubuntu/inventory.ini as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
```

### Cause
The `ansible ping` command was run from the home directory (`~`) but the `inventory.ini` file was located inside `~/DWA/`.

### Fix
Either navigate to the correct directory first:
```bash
cd ~/DWA/
ansible -i inventory.ini all -m ping
```
Or specify the full path to the inventory file:
```bash
ansible -i /home/ubuntu/DWA/inventory.ini all -m ping
```

---

## 4. Ansible SSH - No Such Identity / Permission Denied

### Error
```
Failed to connect to the host via ssh: no such identity: /home/ubuntu/.ssh/Control-node.pem: 
No such file or directory
ubuntu@172.31.73.243: Permission denied (publickey).
```

### Cause
The `.pem` file path in the inventory file was incorrect. The file was referenced as `~/home/ubuntu/.ssh/Control-node.pem` which mixed `~` and the full path together.

### Fix
Corrected the inventory file to use the proper path:
```ini
[webserver]
172.31.73.243 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/Control-node.pem
```

---

## 5. Ansible SSH - Wrong Variable Name in Inventory

### Error
```
Permission denied (publickey)
```

### Cause
The inventory file used incorrect Ansible variable names:
- `ansible_private_key_file` (missing `_ssh_`)
- `ansible_private_ssh_key_file` (wrong word order)

### Fix
Used the correct Ansible variable name:
```ini
ansible_ssh_private_key_file=/home/ubuntu/.ssh/control-node.pem
```

---

## 6. Ansible Playbook - UNREACHABLE (Wrong ansible_user)

### Error
```
fatal: [172.31.73.243]: UNREACHABLE! => {
    "msg": "Failed to connect to the host via ssh: ansibleuser@172.31.73.243: Permission denied (publickey)."
}
```

### Cause
The inventory file had `ansible_user=ansibleuser` but `ansibleuser` did not exist yet on the Remote-node. This was a chicken-and-egg problem — Ansible needs to connect first to create the user.

### Fix
Changed `ansible_user` back to `ubuntu` (the default EC2 user) for the initial connection:
```ini
[webserver]
172.31.73.243 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/control-node.pem
```
Ansible then connected as `ubuntu` and created `ansibleuser` as defined in the playbook tasks.

---

## 7. Reserved Variable Name Warning

### Warning
```
[WARNING]: Found variable using reserved name: port
```

### Cause
`port` is a reserved keyword in Ansible and cannot be used as a custom variable name.

### Fix
Renamed the variable from `port` to `port1` in both:
- `playbook.yml` vars section
- `templates/nginx.conf.j2` placeholder

**playbook.yml:**
```yaml
vars:
  port1: 80
```

**nginx.conf.j2:**
```nginx
listen {{ port1 }};
```

---

## 8. Website Not Accessible in Browser

### Error
```
This site can't be reached. 3.215.183.99 took too long to respond.
```

### Cause
Port 80 (HTTP) was not open in the AWS Security Group inbound rules for the Remote-node.

### Fix
Added the following inbound rule to the Remote-node's Security Group:
- **Type:** HTTP
- **Port:** 80
- **Source:** `0.0.0.0/0`

After adding the rule, the website was successfully accessible at `http://<Remote-node-public-IP>`.

---

## 9. Git Push - Repository Not Found

### Error
```
remote: Repository not found.
fatal: repository 'https://github.com/YOUR_USERNAME/ansible-web-deployment.git/' not found
```

### Cause
The git remote was still pointing to the placeholder URL `YOUR_USERNAME` instead of the actual GitHub username.

### Fix
Removed the incorrect remote and added the correct one:
```bash
git remote remove origin
git remote add origin https://github.com/sukhvindar-glitch/ansible-web-deployment.git
git push -u origin main
```

---

## Key Learnings

- Always verify file paths before running commands
- SSH private keys must have `chmod 400` permissions
- Ansible variable names are case-sensitive and some names are reserved
- Security Groups must explicitly allow inbound traffic on required ports
- The `ansible_user` in inventory must be an existing user on the remote node for initial connection
- Use `ansible -i inventory.ini all -m ping` to test connectivity before running playbooks
