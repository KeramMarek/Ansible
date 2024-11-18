```
ansible --version -> shows version and basic paths.
```

### Inventory file
```
vim hosts.
```
```
ubuntu ansible_connection=local ansible_user=ubuntu VARIABLE=VALUE
rhel ansible_host=54.93.174.83 ansible_port=22 ansible_user=ec2-user
debian ansible_host=3.71.12.45 ansible_port=22 ansible_user=admin

# {name} ansible_host={remote_host_IP} ansible_port={port} ansible_user={remote_host_user} VARIABLE=VALUE #
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
export ANSIBLE_CONFIG=/path/to/config -> create variable, you can override restrictions with this. Like you can store hosts file even in /tmp.
unset ANSIBLE_CONFIG -> unset variable
```
```
vim .ansible.cfg
```
```
[defaults] -> set defaults for Ansible.
inventory=/tmp/hosts -> set path to inventory.
```
```
ansible {host_name}/all -m setup -> info about node.
```
```
ansible {host/group} -m {ansible_module} -a {arguments} -b
-m -> stands for module.
-a -> stands for arguments.
-b -> stands for become, like root.
-i -> if you want to specify hosts file and not take it from inventory.
--ask-become-pass -> ask for password
```
```
Always search for module when you need something, if there is no module then use command/shell. Read the doc.
```
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
--ask-become-pass -> ask for password for sudo.
ansible dbservers -m command -a "/sbin/reboot" -b -> reboot.
ansible webservers -m command -a "hostname" -b -> execute command hostname on remote host.

```
### Playbook
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
```
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


ansible-playbook {playbook.yaml} -> execute your playbook.
--tag {tag} -> execute only task with certain task.



