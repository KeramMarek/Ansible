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
   - [Add Repository and GPG key](#add-repository-and-gpg-key)
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
   - [Block](#block)
   - [When](#when)
   - [Include](#include)
   - [Example Handler](#example-handler)
   - [Dependencies](#dependencies)
8. [Register](#register)
   - [Using Register](#using-register)
9. [Gather Facts](#gather-facts)
   - [When to Use `gather_facts`](#when-to-use-gather_facts)
10. [Example Playbooks](#example-playbooks)

---

## Ansible Basics
### Installing ansible:
```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```
Using pip (cross-platform, including Windows via WSL or Linux):
```bash
pip install ansible
python3 -m venv ansible-env
source ansible-env/bin/activate
pip install ansible
```
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
ubuntu ansible_connection=local ansible_user=ubuntu ansible_become_pass='{{ ubuntu_sudo_pass }}' -> for local connection see diff between ubuntu and rhel.

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

# Variables for All
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
ansible redhat -m file -a "path=/tmp/test state=directory"
```

#### Helpful Command:
```bash
ansible <host> -m ansible.builtin.setup
```
- Displays host information and available variables.

---

## Playbooks

### Basics

ALWAYS TRY TO LOOK FOR ANSIBLE MODULE FOR WHATEVER YOU NEED. Copy, download, apt etc.

Run a playbook:
```bash
ansible-playbook {playbook.yaml} -> execute your playbook.
--tag {tag} -> execute only task with certain task.
-e @pass.yaml -> use your passwords file.
--ask-vault-pass -> ask for vault password.
--ask-become-pass -> ask for sudo password.
-check -> starts dry run.
```

### Add Repository and GPG key

```yaml
- name: Add PHP GPG key
  become: yes
  apt_key:
    url: https://packages.sury.org/php/apt.gpg
    state: present

- name: Add PHP repository for Debian
  become: yes
  apt_repository:
    repo: "deb https://packages.sury.org/php/ {{ ansible_distribution_release }} main"
    state: present
```

```
This was in documentation for GPG and Repo

wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list 

$(lsb_release -sc is changed by {{ ansible_distribution_release }} for function. 
You can see output of $(lsb_release -sc) and find in ansible <host> -m ansible.builtin.setup variable with same output which is {{ ansible_distribution_release }}.
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
```yaml

vim common.yml

- name: Common
  hosts:
    - ubuntu
  #gather_facts: no
  tasks:
  - name: Install all packages
    become: yes
    apt:
      name:
        - htop
        - vim
        - kazam
        - filezilla
        - bluefish
      state: present
      update_cache: yes
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
```
- everything in loop is considerd as item so that you can reach to loop.
- in variables we have servers and name.
- we can use that as {{ item.name }}.
- looping through {{ servers }} means every loop take on item.
- if you put only {{ item }} you will get back {'name': 'rhel'}.

So in example up item = servers, item.name = ubuntu ...
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

### Example Password File created by ansible-vault create pass.yaml 

#### File: `pass.yaml`
```yaml
ubuntu_sudo_pass: my_password
```

- add entry to your inventory file ansible_become_pass='{{ ubuntu_sudo_pass }}
- {{ ubuntu_sudo_pass }} -> this variable we specifed in pass.yaml
  
#### Inventory File Example: 
```ini
ubuntu ansible_connection=local ansible_user=ubuntu ansible_become_pass='{{ ubuntu_sudo_pass }}'
```

#### Run with Vault:
```bash
ansible-playbook {playbook.yaml} -e @pass.yaml --ask-vault-pass
```
#### Run without --ask-vault-pass:
```
If you don't want to be asked for password to vault then do following:
Create .vault_pass and add your password there:
.vault_pass
password
Add entry to your ansible.cfg:
ansible.cfg
[defaults]
vault_password_file=.vault_pass
```
#### Run without -e @pass.yaml:
```
If you don't want to use -e @pass.yaml u can use in your playbook:
vars_files:
    - passwds.yml
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

### Example Role
```yaml
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

### Block

Key Benefits

    Error Handling:
        Perform alternate actions when a set of tasks fails.
    Cleanup:
        Always execute cleanup tasks regardless of task success or failure.
    Code Readability:
        Logically groups tasks together, making playbooks easier to understand and maintain.

Using block is particularly useful when you have complex playbooks that involve dependent tasks or need to ensure cleanup after a failure.

```yaml
- name: Install and verify package with block
  hosts: localhost
  tasks:
    - name: Ensure nginx is installed
      block:
        - name: Install nginx
          yum:
            name: nginx
            state: present

        - name: Verify nginx version
          command: nginx -v
          register: nginx_version
          failed_when: "'command not found' in nginx_version.stderr"

      rescue:
        - name: Log error
          debug:
            msg: "Failed to install or verify nginx"

      always:
        - name: Cleanup temporary files
          file:
            path: /tmp/nginx_install_log
            state: absent
```

---

### When

Option 1: Using and to Combine Conditions
```yaml
- include_tasks: ubuntu.yml
  when: ansible_lsb.id == 'Ubuntu' and ansible_os_family != 'Debian'
```
Option 2: Using a List for Multiple Conditions
```yaml
- include_tasks: ubuntu.yml
  when:
    - ansible_lsb.id == 'Ubuntu'
    - ansible_os_family != 'Debian'
```
Examples:
```yaml
- name: Install Apache on RedHat-based systems
  yum:
    name: httpd
    state: present
  when: ansible_os_family == "RedHat"
```
```yaml
- name: Install specific package on Ubuntu 20.04
  apt:
    name: mypackage
    state: present
  when: ansible_distribution == "Ubuntu" and ansible_distribution_version == "20.04"
```
```yaml
- name: Restart a service if a variable is true
  service:
    name: apache2
    state: restarted
  when: restart_apache is defined and restart_apache
```
```yaml
- name: Copy a file only if it doesn't exist
  copy:
    src: /path/to/source/file
    dest: /path/to/destination/file
  when: not file_exists.stat.exists
  register: file_exists
  delegate_to: localhost
```

---

### Include

You can create in tasks debian.yml and ubuntu.yml and in main.yml copy include_tasks which checks condition and run only yml which meets the condition.

```yaml
- include_tasks: debian.yml
  when: ansible_os_family == 'Debian'

- include_tasks: ubuntu.yml
  when: ansible_os_family == 'Ubuntu'
```

---

### Example Handler

Call handler with notify:

Example:
#### File: `tasks/main.yml`
```yaml
- name: Install Apache
  apt:
    name: apache2
    state: present
  notify: restart_apache
```
#### File: `handlers/main.yml`
```yaml
- name: restart_apache
  service:
    name: apache2
    state: restarted
```

---

### Dependencies
In meta folder you can add to dependencies role to execute.
```yaml
dependencies:
	- lamp
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

---

# Example playbooks

```yaml
- name: Create and execute script.sh
  hosts:
    - ubuntu

  tasks:
    - name: Create script.sh
      shell: |
        echo "#!/bin/bash" > script.sh
        echo "touch emptyfile.txt" >> script.sh
        echo "echo 'emptyfile.txt has been created'" >> script.sh
        chmod +x script.sh

    - name: Execute script.sh
      shell: ./script.sh > output.txt

    - name: Cleaning
      command: rm -rf script.sh emptyfile.txt
```

```yaml
- name: Install Apache
  hosts:
    - ubuntu
  gather_facts: no

  tasks:
    - name: Install Apache
      become: yes
      ansible.builtin.apt:
        name:
          - apache2
        state: present

    - name: Ensure the default Apache port is 8080
      become: yes
      ansible.builtin.lineinfile:
        path: /etc/apache2/apache2.conf
        regexp: '^Listen '
        insertafter: '^#Listen '
        line: Listen 8080

    - name: Copy file with owner and permissions
      become: yes
      ansible.builtin.copy:
        src: ~/ansible/index.html
        dest: /var/www/html/index.html
        owner: ubuntu
        group: ubuntu
        mode: '0644'
```
# Full scrpt task:
```yaml
- name: creating a script
  hosts:
    - ubuntu

  tasks:
      # - name: created script.sh
      #   shell: |
      #     echo "#!/bin/bash" > script.sh
      #     echo "touch emptyfile.txt" >> script.sh
      #     echo "echo 'emptyfile.txt has been created'" >> script.sh
      #     chmod +x script.sh

      # - name: Execute script.sh
      #   shell: ./script.sh > output.txt

      
        - name: Create an empty file
          file:
            path: /home/ubuntu/marek.grohol/main_practice/script.sh
            state: touch

        - name: fill the script
          shell: |
            echo "#!/bin/bash" > script.sh
            echo "touch emptyfile.txt" >> script.sh
            echo "echo 'empty file has been created'" >> script.sh
            chmod +x script.sh

        - name: execute the script
          shell:
            ./script.sh > output.txt
        
        - name: Remove two files
          file:
            path: "{{ item }}"
            state: absent
          loop:
            - /home/ubuntu/marek.grohol/main_practice/script.sh
            - /home/ubuntu/marek.grohol/main_practice/emptyfile.txt
            - /home/ubuntu/marek.grohol/main_practice/output.txt
```

# script with lineinfile:
```yaml
- name: Create and execute script.sh
  hosts:
    - ubuntu

  tasks:
    - name: Create script.sh
      shell: |
        echo "#!/bin/bash" > script.sh
        echo "touch emptyfile.txt" >> script.sh
        echo "echo 'emptyfile.txt has been created'" >> script.sh
        chmod +x script.sh

    


    - name: Replace line containing 'created' with 'done'
      lineinfile:
        path: script.sh
        regexp: "created"
        line: "echo 'emptyfile.txt has been done'"

    - name: execut
      shell: ./script.sh > output.txt
```
