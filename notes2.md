# Ansible Notes

## Table of Contents

1. [Ansible Basics](#ansible-basics)
2. [Configuration Files](#configuration-files)
   - [Inventory File (`hosts`)](#inventory-file-hosts)
   - [Configuration File (`ansible.cfg`)](#configuration-file-ansiblecfg)
3. [Ansible Commands](#ansible-commands)
   - [Command Structure](#command-structure)
   - [Examples](#examples)
4. [Playbooks](#playbooks)
   - [Basics](#basics)
   - [Example Playbook](#example-playbook)
5. [Templates](#templates)
   - [File Example](#file-example)
   - [Playbook Using Template](#playbook-using-template)
6. [Password Management](#password-management)
   - [Encrypting Passwords](#encrypting-passwords)
   - [Example Password File](#example-password-file)
7. [Roles](#roles)
   - [Initializing a Role](#initializing-a-role)
   - [Directory Structure](#directory-structure)
   - [Example Handler](#example-handler)
8. [Register](#register)
   - [Using Register](#using-register)
9. [Gather Facts](#gather-facts)
   - [When to Use `gather_facts`](#when-to-use-gather_facts)

---

## Ansible Basics

### Check Version
```bash
ansible --version
```
- Displays Ansible version and basic paths.

---

## Configuration Files

### Inventory File (`hosts`)

The inventory file specifies hosts and their properties.

#### Example:
```ini
# Define hosts
ubuntu ansible_connection=local ansible_user=ubuntu ansible_become_pass='{{ ubuntu_sudo_pass }}'
rhel ansible_host=54.93.174.83 ansible_port=22 ansible_user=ec2-user
debian ansible_host=3.71.12.45 ansible_port=22 ansible_user=admin

# Groups
[webservers]
rhel
debian

[webservers2]
ubuntu

# Group Variables
[webservers2:vars]
ansible_user=ubuntu

[all:vars]
ansible_user=ubuntu
```

#### Key Syntax:
```ini
{name} ansible_host={IP} ansible_port={port} ansible_user={user} ansible_become_pass='{{ value }}'
```

---

### Configuration File (`ansible.cfg`)

The main configuration file sets global defaults.

#### Example:
```ini
[defaults]
inventory=/tmp/hosts          # Path to inventory file
vault_password_file=.vault_pass  # Path to vault password
```

#### Environment Overrides:
```bash
export ANSIBLE_CONFIG=/path/to/config
unset ANSIBLE_CONFIG
```

---

## Ansible Commands

### Command Structure

```bash
ansible {host/group} -m {module} -a {arguments} -b
```

#### Options:
- `-m`: Specify module.
- `-a`: Specify arguments.
- `-b`: Use become (e.g., root).
- `-i`: Specify inventory file (if not using default).
- `--ask-become-pass`: Prompt for sudo password.

---

### Examples

```bash
ansible debian -m apt -a "name=cmatrix" -b         # Install package
ansible debian -m command -a "mkdir /tmp/nove" -b  # Create directory
ansible webservers -m hostname -a "name=new-hostname" -b  # Change hostname
ansible debian -m copy -a "src=/home/ubuntu/test dest=/tmp" -b  # Copy file
ansible dbservers -m reboot -b --ask-become-pass   # Reboot and wait
```

#### Helpful Command:
```bash
ansible <host> -m ansible.builtin.setup
```
- Displays host information and available variables.

---

## Playbooks

### Basics

Run a playbook:
```bash
ansible-playbook {playbook.yaml} -> execute your playbook.
--tag {tag} -> execute only task with certain task.
-e @pass.yaml -> use your passwords file.
--ask-vault-pass -> ask for vault password.
--ask-become-pass -> ask for sudo password.
```

---

### Example Playbook

#### File: `playbook.yml`
```yaml
- name: Example Playbook
  hosts:
    - ubuntu
    - debian
    - rhel
  gather_facts: no

  vars:
    my_var: some_value
  vars_file:
    - path

  tasks:
    - name: Install cmatrix
      become: yes
      ansible.builtin.apt:
        name: cmatrix
        state: present
        update_cache: yes
      tags:
        - install

    - name: Debug Variable
      debug:
        var: my_var
```

---

## Templates

### File Example

#### Template File: `host.j2`
```jinja
This is file for host: {{ item.name }}
```

---

### Playbook Using Template

```yaml
- name: Template Example
  hosts:
    - ubuntu
  gather_facts: yes

  vars:
    servers:
      - name: ubuntu
      - name: debian

  tasks:
    - name: Generate Host Files
      template:
        src: host.j2
        dest: "/tmp/{{ item.name }}.txt"
      loop: "{{ servers }}"
      tags:
        - template
```

---

## Password Management

### Encrypting Passwords

```bash
ansible-vault create pass.yaml      # Create encrypted file
ansible-vault encrypt pass.yaml     # Encrypt file
ansible-vault decrypt pass.yaml     # Decrypt file
ansible-vault edit pass.yaml        # Edit encrypted file
```

---

### Example Password File

#### File: `pass.yaml`
```yaml
ubuntu_sudo_pass: my_password
```

#### Inventory File Example:
```ini
ubuntu ansible_connection=local ansible_user=ubuntu ansible_become_pass='{{ ubuntu_sudo_pass }}'
```

#### Run with Vault:
```bash
ansible-playbook {playbook.yaml} -e @pass.yaml --ask-vault-pass
```

---

## Roles

### Initializing a Role

```bash
ansible-galaxy init my_role
```

---

### Directory Structure

```
my_role/
├── defaults/        # Default variables
├── files/           # Static files
├── handlers/        # Handlers go here
│   └── main.yml
├── tasks/           # Tasks for the role
├── templates/       # Templates (.j2 files)
├── vars/            # Variables for the role
└── meta/            # Metadata
```

### Example Role
```yaml
---
# tasks file for dev
- name: Install git and docker prerequisities
  become: yes
  apt:
    name: 
      - git
      - ca-certificates
      - curl
    state: present
    update_cache: yes
    cache_valid_time: 600
  tags:
    - dev

- name: "Create {{ keyrings_dir }} dir"
  become: yes
  file:
    name: "{{ keyrings_dir }}"
    state: directory
    mode: 0755

- name: Add GPG key for docker repository
  become: yes
  ansible.builtin.apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    keyring: "{{ keyrings_dir }}/docker.gpg"
    state: present

- name: "Set correct rights for {{ keyrings_dir }}/docker.gpg"
  become: yes
  file:
    path: "{{ keyrings_dir }}/docker.gpg"
    mode: a+r

- name: Get OS arch
  command: dpkg --print-architecture
  register: arch_out

- name: Add the repository to Apt sources" -> apt module
  become: yes
  ansible.builtin.apt_repository:
    repo: "deb [arch={{ arch_out.stdout }} signed-by={{ keyrings_dir }}/docker.gpg] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    filename: docker
    state: present

- name:  Add the repository to Apt sources" -> shell
  become: yes
  shell: | -> this pipeline tells that the command will be on more lines.
    "deb [arch=$(dpkg --print-architecture) signed-by={{ keyrings_dir }}/docker.gpg] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    tee /etc/apt/sources.list.d/docker.list > /dev/null

- name: Install docker
  become: yes
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present
    update_cache: yes
  notify: start docker service

- name: Ensure group "docker" exists
  become: yes
  ansible.builtin.group:
    name: docker
    state: present

- name: Add the users to docker group
  become: yes
  ansible.builtin.user:
    name: "{{ item }}"
    groups: docker
    append: yes
  loop: "{{ docker_users }}"
```

---

### Example Handler

#### File: `handlers/main.yml`
```yaml
- name: restart_apache
  service:
    name: apache2
    state: restarted
```

#### File: `tasks/main.yml`
```yaml
- name: Install Apache
  apt:
    name: apache2
    state: present
  notify: restart_apache
```

---

## Register

### Using Register

```yaml
- name: Register Example
  command: echo "hello"
  register: command_output

- debug:
    var: command_output.stdout -> .stdout because you can see lower, that "ahoj" is in stdout.
```

```yaml
command_output:
ok: [ubuntu] => {
    "arch_out": {
        "changed": true,
        "cmd": [
            "echo"
            "ahoj"
        ],
        "delta": "0:00:00.004797",
        "end": "2024-11-20 09:38:57.256258",
        "failed": false,
        "msg": "",
        "rc": 0,
        "start": "2024-11-20 09:38:57.251461",
        "stderr": "",
        "stderr_lines": [],
        "stdout": "ahoj",    -----> here is value that we need.
        "stdout_lines": [
            "ahoj"
        ]
    }
}
```

---

## Gather Facts

### When to Use `gather_facts`

Gathering facts collects system details like:
- OS (`ansible_os_family`)
- Network (`ansible_default_ipv4`)
- CPU and memory.

#### Example:
```yaml
- name: Install Packages Based on OS
  tasks:
    - name: Debian-based
      apt:
        name: apache2
        state: present
      when: ansible_os_family == "Debian"

    - name: RedHat-based
      yum:
        name: httpd
        state: present
      when: ansible_os_family == "RedHat"
```
