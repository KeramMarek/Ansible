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
ansible-playbook {playbook.yaml} --tag {tag} -e @pass.yaml --ask-vault-pass --ask-become-pass
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
    var: command_output.stdout
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
