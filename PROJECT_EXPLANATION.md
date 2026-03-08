# Ansible Web Deployment - Project Architecture & Code Explanation

---

## 1. Project Structure Overview

```
~/DWA/
├── inventory.ini          ← Tells Ansible WHERE to connect
├── playbook.yml           ← Tells Ansible WHAT to do
├── files/
│   └── index.html         ← The actual website content
└── template/
    └── nginx.conf.j2      ← Blueprint for Nginx configuration
```

### How All Files Connect

```
inventory.ini  ──────────────────────────────────────────┐
                                                          ▼
index.html  ──────────────────────────────────────► playbook.yml ──► Remote-node
                                                          ▲
nginx.conf.j2  ──────────────────────────────────────────┘
```

- **inventory.ini** tells the playbook WHERE to deploy (which server, which user, which key)
- **index.html** is the website file the playbook COPIES to the remote server
- **nginx.conf.j2** is the template the playbook uses to CONFIGURE Nginx
- **playbook.yml** is the BRAIN that orchestrates everything

---

## 2. File-by-File Explanation

---

### 2.1 inventory.ini

#### Why Was It Created?
Ansible needs to know which remote servers to manage. The inventory file is like a **contact list** — it tells Ansible:
- Where the server is (IP address)
- How to authenticate (SSH key)
- Which user to connect as

#### The Code
```ini
[webserver]
172.31.73.243 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/control-node.pem
```

#### Code Breakdown

| Part | Meaning |
|------|---------|
| `[webserver]` | Group name — organizes servers into logical groups. The playbook references this group name in `hosts:` |
| `172.31.73.243` | Private IP address of the Remote-node (EC2 instance) |
| `ansible_user=ubuntu` | The SSH username Ansible uses to connect — `ubuntu` is the default user on Ubuntu EC2 instances |
| `ansible_ssh_private_key_file=...` | Full path to the `.pem` private key file used for SSH authentication |

#### How It Connects to playbook.yml
In the playbook, this line references the inventory group:
```yaml
hosts: webserver   ← matches [webserver] in inventory.ini
```
Without the inventory file, Ansible wouldn't know which server to connect to.

---

### 2.2 nginx.conf.j2 (Nginx Configuration Template)

#### Why Was It Created?
Instead of hardcoding values like port number, server name, and document root directly into the Nginx config, we use a **Jinja2 template** with placeholders. This makes the configuration:
- **Dynamic** — values come from playbook variables
- **Reusable** — same template works for different servers just by changing variables
- **Scalable** — deploy to 10 servers with different configs without changing the template

#### The Code
```nginx
server {
    listen {{ port1 }};
    server_name {{ server_name }};
    root {{ doc_root }};
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### Code Breakdown

| Part | Meaning |
|------|---------|
| `server { }` | Nginx server block — defines configuration for one virtual server |
| `listen {{ port1 }}` | Port Nginx listens on. `{{ port1 }}` is replaced by the value of `port1` variable from playbook (80) |
| `server_name {{ server_name }}` | Domain or IP of the server. Replaced by `server_name` variable (172.31.73.243) |
| `root {{ doc_root }}` | Directory where website files are served from. Replaced by `doc_root` variable (/var/www/html/) |
| `index index.html index.htm` | Default file to serve when someone visits the root URL |
| `location / { }` | Handles all URL requests to the server |
| `try_files $uri $uri/ =404` | Try to find the requested file; if not found return 404 error |
| `{{ }}` | Jinja2 placeholder syntax — Ansible replaces these with actual variable values at runtime |

#### How It Connects to playbook.yml
The playbook uses the `template` module to:
1. Read `nginx.conf.j2` from the Control-node
2. Replace all `{{ }}` placeholders with values from `vars`
3. Copy the generated config file to `/etc/nginx/sites-available/myapp` on the Remote-node

```yaml
vars:
  port1: 80               ──► replaces {{ port1 }}
  server_name: 172.31.73.243  ──► replaces {{ server_name }}
  doc_root: /var/www/html/    ──► replaces {{ doc_root }}
```

---

### 2.3 index.html (Website File)

#### Why Was It Created?
This is the actual webpage that gets served by Nginx. It is the content that users see when they visit the website in their browser.

#### The Code
```html
<!DOCTYPE html>
<html>
<head>
    <title>My Web App</title>
</head>
<body>
    <h1>Hello from Ansible!</h1>
    <p>This page was deployed using Ansible on Nginx!</p>
</body>
</html>
```

#### Code Breakdown

| Part | Meaning |
|------|---------|
| `<!DOCTYPE html>` | Declares this is an HTML5 document |
| `<html>` | Root element of the webpage |
| `<head>` | Contains metadata — not visible on the page |
| `<title>` | Text shown in the browser tab |
| `<body>` | Contains visible content |
| `<h1>` | Main heading displayed on the page |
| `<p>` | Paragraph of text |

#### How It Connects to playbook.yml
The playbook uses the `copy` module to copy this file from the Control-node to the Remote-node:
```yaml
- name: Copy file with owner and permissions
  ansible.builtin.copy:
    src: /home/ubuntu/DWA/files/     ← source on Control-node
    dest: /var/www/html/             ← destination on Remote-node
```
After copying, Nginx serves this file when someone visits the website.

---

### 2.4 playbook.yml (The Brain)

#### Why Was It Created?
The playbook is the **central automation script** that orchestrates everything. It defines what tasks to run, in what order, on which servers.

#### The Complete Code
```yaml
---
- name: project1
  hosts: webserver
  become: yes

  vars:
    port1: 80
    server_name: 172.31.73.243
    doc_root: /var/www/html/

  tasks:
    - name: Create a user 'ansibleuser' with a home directory
      ansible.builtin.user:
        name: ansibleuser
        create_home: yes
        groups: sudo

    - name: Install "nginx"
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Copy file with owner and permissions
      ansible.builtin.copy:
        src: /home/ubuntu/DWA/files/
        dest: /var/www/html/
        mode: '0644'

    - name: Apply Nginx template
      ansible.builtin.template:
        src: /home/ubuntu/DWA/template/nginx.conf.j2
        dest: /etc/nginx/sites-available/myapp
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

#### Code Breakdown

**Play Header:**
```yaml
- name: project1       # Name of this play
  hosts: webserver     # Target the [webserver] group from inventory.ini
  become: yes          # Use sudo privileges for all tasks
```

**Variables Section:**
```yaml
vars:
  port1: 80                    # HTTP port
  server_name: 172.31.73.243   # Remote-node IP
  doc_root: /var/www/html/     # Nginx document root
```
These variables are injected into `nginx.conf.j2` template at runtime.

**Task 1 - Create User:**
```yaml
- name: Create a user 'ansibleuser' with a home directory
  ansible.builtin.user:
    name: ansibleuser      # Username to create
    create_home: yes       # Create home directory /home/ansibleuser
    groups: sudo           # Add to sudo group for admin privileges
```

**Task 2 - Install Nginx:**
```yaml
- name: Install "nginx"
  ansible.builtin.apt:
    name: nginx        # Package to install
    state: present     # Ensure it is installed (idempotent)
    update_cache: yes  # Run apt update before installing
```

**Task 3 - Copy Website Files:**
```yaml
- name: Copy file with owner and permissions
  ansible.builtin.copy:
    src: /home/ubuntu/DWA/files/   # Source on Control-node
    dest: /var/www/html/           # Destination on Remote-node
    mode: '0644'                   # File permissions (owner read/write, others read)
```

**Task 4 - Apply Nginx Template:**
```yaml
- name: Apply Nginx template
  ansible.builtin.template:
    src: /home/ubuntu/DWA/template/nginx.conf.j2   # Template on Control-node
    dest: /etc/nginx/sites-available/myapp          # Generated config on Remote-node
  notify: Restart nginx   # Trigger handler if this task makes a change
```

**Handler - Restart Nginx:**
```yaml
handlers:
  - name: Restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted   # Restart Nginx to apply new configuration
```
The handler only runs when notified — meaning Nginx only restarts when the config actually changes.

---

## 3. What Happens When You Run the Playbook

### Command
```bash
ansible-playbook playbook.yml -i inventory.ini
```

### Step-by-Step Background Execution

```
STEP 1: PARSE & VALIDATE
ansible-playbook reads playbook.yml and inventory.ini
↓
Validates YAML syntax
Loads variable values from vars section
Identifies target hosts from inventory.ini

STEP 2: GATHER FACTS
Ansible SSHes into Remote-node as ubuntu using control-node.pem
↓
Collects system information (OS, IP, memory, etc.)
This info can be used in tasks if needed

STEP 3: TASK 1 - Create User
Ansible runs the user module on Remote-node
↓
Checks if ansibleuser already exists
If not → creates user with home directory and adds to sudo group
If yes → skips (idempotency)

STEP 4: TASK 2 - Install Nginx
Ansible runs apt update on Remote-node
↓
Checks if nginx is already installed
If not → installs nginx
If yes → skips (idempotency)

STEP 5: TASK 3 - Copy Website Files
Ansible reads index.html from Control-node
↓
Transfers the file over SSH to /var/www/html/ on Remote-node
Sets file permissions to 0644

STEP 6: TASK 4 - Apply Nginx Template
Ansible reads nginx.conf.j2 from Control-node
↓
Replaces {{ port1 }} → 80
Replaces {{ server_name }} → 172.31.73.243
Replaces {{ doc_root }} → /var/www/html/
↓
Copies generated config to /etc/nginx/sites-available/myapp on Remote-node
Notifies "Restart nginx" handler because config changed

STEP 7: HANDLER - Restart Nginx
Handler was notified in Step 6
↓
Restarts Nginx service on Remote-node
New configuration takes effect

STEP 8: PLAY RECAP
Ansible prints summary:
ok=6  changed=5  unreachable=0  failed=0
```

---

## 4. How All Files Work Together - The Big Picture

```
                    CONTROL-NODE
                    ┌─────────────────────────────────┐
                    │                                 │
                    │  inventory.ini ─────────────────┼──► SSH connection details
                    │                                 │
                    │  playbook.yml ──────────────────┼──► Orchestrates everything
                    │       │                         │
                    │       ├── reads vars            │
                    │       ├── reads inventory.ini   │
                    │       ├── reads index.html      │
                    │       └── reads nginx.conf.j2   │
                    │                                 │
                    │  files/index.html               │
                    │  template/nginx.conf.j2         │
                    │                                 │
                    └──────────────┬──────────────────┘
                                   │ SSH (port 22)
                                   │ using control-node.pem
                                   ▼
                    REMOTE-NODE
                    ┌─────────────────────────────────┐
                    │                                 │
                    │  Creates: ansibleuser           │
                    │  Installs: nginx                │
                    │  /var/www/html/index.html  ◄────┼── copied from Control-node
                    │  /etc/nginx/sites-available/ ◄──┼── generated from template
                    │                                 │
                    └──────────────┬──────────────────┘
                                   │ HTTP (port 80)
                                   ▼
                    BROWSER
                    ┌─────────────────────────────────┐
                    │  http://3.215.183.99             │
                    │  Hello from Ansible!             │
                    └─────────────────────────────────┘
```

---

## 5. Key Concepts Summary

| Concept | Description |
|---------|-------------|
| **Idempotency** | If a task is already done, Ansible skips it. Running the playbook 10 times gives the same result as running it once |
| **Handlers** | Special tasks that only run when notified — Nginx only restarts when config actually changes |
| **Templates** | Dynamic config files using Jinja2 `{{ }}` placeholders replaced by variable values at runtime |
| **Modules** | Built-in Ansible tools for specific tasks — `apt`, `copy`, `template`, `user`, `service` |
| **Become** | Tells Ansible to use sudo for tasks that require root privileges |
| **Agentless** | Ansible only needs to be installed on the Control-node — Remote-node just needs SSH |
