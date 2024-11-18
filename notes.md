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
nsible debian -m command -a "mkdir /tmp/nove" -b -> create dir.
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


