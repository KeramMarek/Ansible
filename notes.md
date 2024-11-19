```

ansible --version -> shows version and basic paths.

```

### Inventory file

```

vim hosts

ubuntu ansible_connection=local ansible_user=ubuntu ansible_become_pass='{{ ubuntu_sudo_pass }}'
rhel ansible_host=54.93.174.83 ansible_port=22 ansible_user=ec2-user
debian ansible_host=3.71.12.45 ansible_port=22 ansible_user=admin

# {name} ansible_host={remote_host_IP} ansible_port={port} ansible_user={remote_host_user} ansible_become_pass='{{ value }}' #
# for local host: #
# {name} ansible_connection=local ansible_user={local_host_user} VARIABLE=VALUE #

[webservers] -> create group from hosts. Names are from settings up.
rhel
debian

[webservers2] -> create group from hosts. Names are from settings up.
ubuntu

[webservers2:vars] -> group variables.
ansible_user=ubuntu

[all:vars] -> variables for all.
ansible_user=ubuntu

```

### Config file

```

vim ansible.cfg

[defaults] -> set defaults for Ansible.
inventory=/tmp/hosts -> set path to inventory.
vault_password_file=.vault_pass -> path to vault pass.

```

### Ansible CLI commands

```

ansible {host/group} -m {ansible_module} -a {arguments} -b
-m -> stands for module.
-a -> stands for arguments.
-b -> stands for become, like root.
-i -> if you want to specify hosts file and not take it from inventory.
--ask-become-pass -> ask for sudo password.

```

## Always search for module when you need something, if there is no module then use command/shell. Read the doc.

```

Examples:

ansible debian -m apt -a "name=cmatrix" -b -> install cmatrix.
ansible debian -m command -a "mkdir /tmp/nove" -b -> create dir.
ansible webservers -m hostname -a "name=martinvyhonsky" -b -> change hostname
ansible debian -m copy -a "src=/home/ubuntu/test dest=/tmp" -b -> copy file.
ansible debian -m command -a "mv /tmp/test /tmp/nove/" -b -> move file.
ansible webservers -m apt -a "name=vim update_cache=yes" -b -> install vim on remote hostname.
ansible dbservers -m shell -a "/sbin/reboot" -b -> reboots and disconnect immiadetely.
ansible dbservers -m reboot -b --ask-become-pass -> wait for reboot response after reboot.
--ask-become-pass -> ask for sudo password.
ansible dbservers -m command -a "/sbin/reboot" -b -> reboot.
ansible webservers -m command -a "hostname" -b -> execute command hostname on remote host
ansible <hostname> -m ansible.builtin.setup -> shows informations about host you can pick up variables from here.

```

```

export ANSIBLE_CONFIG=/path/to/config -> create variable, you can override restrictions with this. Like you can store hosts file even in /tmp.
unset ANSIBLE_CONFIG -> unset variable

```

### Playbook

```

ansible-playbook {playbook.yaml} -> execute your playbook.
--tag {tag} -> execute only task with certain task.
-e @pass.yaml -> use your passwords file.
--ask-vault-pass -> ask for vault password.
--ask-become-pass -> ask for sudo password.

```

```

vim playbook.yml

- name: Testing Playbook
  hosts:
    - ubuntu
    - debian
    - rhel
  gather_facts: no -> won't collect data of host.

  vars:
    moja_var: text
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
        - instalacia

    - name: Testing shell command
      shell: echo test
      tags:
        - shell

    - debug:
        var: moja_var

```

```

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

### Template

```

Everything in loop is considerd as item so that you can reach to loop.
In variables we have servers and name.
We can use that as {{ item.name }}.
Looping through {{ servers }} means every loop take on item.
If you put only {{ item }} you will get back {'name': 'rhel'}.

```

```

vim. host.j2
This is file for host: {{ item.name }}

```

```

- name: CPU playbook
  hosts:
    - ubuntu
    #- debian
    #- rhel
  gather_facts: yes
  vars:
    servers:
      - name: ubuntu
      - name: debian
      - name: rhel
  tasks:
    - name: servers
      template:
        src: host.j2
        dest: "/tmp/{{ item.name }}.txt"
      loop: "{{ servers }}"
      tags:
        - servers

```

```

What is gather_facts?

    gather_facts: yes (default behavior) collects information about the target host, such as:
        Operating system details (ansible_os_family, ansible_distribution)
        Network interfaces (ansible_default_ipv4, ansible_all_ipv6_addresses)
        Disk usage and file systems
        CPU and memory details
        Hostname and FQDN
        And more

This information is stored in variables and can be used in tasks, templates, conditionals, and handlers.

```

```

When to Use gather_facts: yes

    You Need Host Information:
        If your playbook references facts like ansible_os_family, ansible_hostname, or ansible_distribution, you need to gather them first.
        Example: Installing different packages based on the OS.

    tasks:
      - name: Install Apache on Debian-based systems
        ansible.builtin.apt:
          name: apache2
          state: present
        when: ansible_os_family == "Debian"
      - name: Install Apache on RedHat-based systems
        ansible.builtin.yum:
          name: httpd
          state: present
        when: ansible_os_family == "RedHat"

Dynamic Inventory or Environment Customization:

    If your inventory dynamically defines groups or variables based on system properties, gathering facts ensures they are available.

```

### Passwords

```

ansible-vault create pass.yaml -> ecnrypt your password file.
ansible-vault enecrypt pass.yaml -> ecnrypt your password file.
ansible-vault decrypt pass.yaml -> decrypt your password file
ansible-vault edit pass.yaml -> edit encrypted file.

```

```

ansible-vault create pass.yaml

pass.yaml
ubuntu_sudo_pass: password

```

```

Modify your hosts file:

ansible_become_pass='{{ ubuntu_sudo_pass }}' -> ubuntu_sudo_pass name of variable in passwds.yaml

EXAMPLE:
ubuntu ansible_connection=local ansible_user=ubuntu ansible_become_pass='{{ ubuntu_sudo_pass }}'

```

```

You can run playbook with this:

ansible-playbook {playbook.yaml} -e @pass.yaml --ask-vault-pass
-e @pass.yaml -> specify vault file with passwords.
--ask-vault-pass -> ask for password to vault.

```

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

```

If you don't want to use -e @pass.yaml u can use in your playbook:

vars_files:
    - passwds.yml


```
