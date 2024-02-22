# Preparation for the tasks

I will use AWS for the Ansible homework.

Let's configure SSH on the control node to easily manage my managed nodes without SSH warnings and manual host verifications.

First we need to update “~/.ssh/config” file on Control node:

```bash
Host node*
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  User ubuntu
  IdentityFile ~/.ssh/ansible-ssh.pem
```

Second we need to update /etc/hosts for name resolution.
This will allow my VMs to resolve each other's Fully Qualified Domain Names (FQDN) using /etc/hosts. We'll need to update this file on each node with the IPs and hostnames of my instances.

```bash
127.0.0.1 localhost
172.31.26.126 control.example.com control
172.31.25.157 node1.example.com node1
172.31.26.35 node2.example.com node2
```
![alt text](<screenshots/Screenshot 2024-02-20 at 20.15.55.png>)

Now let's verify connectivity:

![alt text](<screenshots/iScreen Shoter - Safari - 240220202007.png>)

![alt text](<screenshots/iScreen Shoter - Safari - 240220202111.png>)

# Task1

1. Install Ansible on the Control Machine

```bash
sudo apt update
sudo apt install ansible -y
```

![alt text](<screenshots/iScreen Shoter - Safari - 240220202538.png>)

2. Using ad hoc commands, do the following tasks on the node1 and node2:

- Test a response from node1 and node2 using the Ansible module ping.

```bash
ansible -i /etc/ansible/hosts -m ping node1.example.com,node2.example.com
```
![alt text](<screenshots/iScreen Shoter - Safari - 240220204624.png>)

- Run the uname -a command on the managed nodes.

```bash
ansible -i /etc/ansible/hosts -m command -a "uname -a" node1.example.com,node2.example.com
```
![alt text](<screenshots/iScreen Shoter - Safari - 240220204832.png>)

- Check uptime.

```bash
ansible -i /etc/ansible/hosts -m command -a "uptime" node1.example.com,node2.example.com
```
![alt text](<screenshots/iScreen Shoter - Safari - 240220204913.png>)

- Install the htop package on the managed nodes.

```bash
ansible -i /etc/ansible/hosts -m apt -a "name=htop state=present" -b node1.example.com,node2.example.com
```
![alt text](<screenshots/iScreen Shoter - Safari - 240220205017.png>)

3. Add the managed nodes to the inventory group managed_nodes. You can use either the default inventory location or create a new inventory file in the location you prefer.

![alt text](<screenshots/iScreen Shoter - Safari - 240220205222.png>)

4. With an ad hoc command, please make sure is everything runs correctly:

- Print the host names of the managed_nodes group from Ansible-facts.

```bash
ansible managed_nodes -i /etc/ansible/hosts -m setup -a "filter=ansible_hostname"
```
![alt text](<screenshots/iScreen Shoter - Safari - 240220205849.png>)

- Print the manage nodes distributive name from Ansible-facts.

```bash
ansible managed_nodes -i /etc/ansible/hosts -m setup -a "filter=ansible_distribution"
```
![alt text](<screenshots/iScreen Shoter - Safari - 240220210128.png>)

5. Create a playbook to print a list of network interfaces for every virtual machine.

```bash
---
- name: List network interfaces
  hosts: managed_nodes
  tasks:
    - name: Gather facts
      ansible.builtin.setup:
        filter: ansible_interfaces

    - name: Print interfaces
      debug:
        var: ansible_interfaces
```

```bash
ansible-playbook -i /etc/ansible/hosts list_interfaces.yml
```

![alt text](<screenshots/iScreen Shoter - Safari - 240220210847.png>)


# Task2

1. Create the new role common with the following structure:
- defaults
- tasks
- handlers

```bash
ansible-galaxy init common
```
![alt text](<screenshots/iScreen Shoter - Safari - 240220212906.png>)

2. Create a task (e.g., a package) to install a list of required packages on all managed nodes. The task must install nothing by default.
(Use include_tasks to include the package tasks in the main.yml file.
The list of required packages includes curl, lsof, mc, nano, tar, unzip, vim, and zip.)

defaults/main.yml (to set default packages to an empty list):

```bash
---
required_packages: []
```

tasks/install_packages.yml (to define the task for installing packages):

```bash
---
- name: Install required packages
  ansible.builtin.apt:
    name: "{{ required_packages }}"
    state: present
  when: required_packages | length > 0
```

tasks/main.yml (to include the package installation task):

```bash
---
- name: Include package installation tasks
  include_tasks: install_packages.yml
```

3. Create a new task to disable SELinux on the managed nodes. The task must do nothing by default. Reboot nodes where SELinux was disabled, and skip reboot if SELinux is already disabled.

tasks/disable_selinux.yml (to check and disable SELinux):

```bash
---
- name: Check SELinux status
  ansible.builtin.command: getenforce
  register: selinux_status
  changed_when: false
  ignore_errors: true

- name: Disable SELinux
  ansible.builtin.command: setenforce 0
  when: selinux_status.stdout == "Enforcing"
  notify: Reboot the system

- name: Set SELinux to be disabled on reboot
  ansible.builtin.lineinfile:
    path: /etc/selinux/config
    regexp: '^SELINUX='
    line: 'SELINUX=disabled'
  when: selinux_status.stdout == "Enforcing"
```

handlers/main.yml (to define the reboot handler):

```bash
---
- name: Reboot the system
  ansible.builtin.reboot:
```

Include the SELinux task in tasks/main.yml:

```bash
- name: Include SELinux disable task
  include_tasks: disable_selinux.yml
```


4. Create a playbook to run the common role. Run the playbook for all managed nodes.

Let's create a playbook named deploy_common.yml:

```bash
---
- name: Deploy common configuration
  hosts: managed_nodes
  become: yes
  roles:
    - common
  vars:
    required_packages:
      - curl
      - lsof
      - mc
      - nano
      - tar
      - unzip
      - vim
      - zip
```

This playbook assigns the common role to all hosts in the managed_nodes group, defines the list of required packages, and ensures tasks are executed with elevated privileges (become: yes).

Let's execute the playbook:

```bash
ansible-playbook -i /path/to/your/inventory deploy_common.yml
```

![alt text](<screenshots/iScreen Shoter - Safari - 240220220345.png>)

![alt text](<screenshots/iScreen Shoter - Safari - 240220220410.png>)

Based on the output, all tasks in the playbook executed except for the SELinux-related tasks, which failed due to the absence of the getenforce command. This likely means SELinux is not installed or enabled on my Ubuntu nodes, as SELinux is commonly used on Red Hat-based distributions like CentOS or RHEL. Ubuntu uses AppArmor instead.


# Task3

1. Create a new role (e.g., collectd) to manage the collectd service (a simple monitoring exporter).

```bash
ansible-galaxy init collectd
```

![alt text](<screenshots/iScreen Shoter - Safari - 240221115547.png>)

- The role must install the latest release of the collectd package and collectd-write_prometheus plugin.

tasks/main.yml:

```bash
---
- name: Install collectd
  ansible.builtin.package:
    name: collectd
    state: latest
  notify: restart collectd

- name: Configure write_prometheus plugin
  ansible.builtin.template:
    src: prometheus.conf.j2
    dest: /etc/collectd/collectd.conf.d/prometheus.conf
  notify: restart collectd

- name: Ensure collectd is enabled and started
  ansible.builtin.service:
    name: collectd
    state: started
    enabled: true
```

- Add a handler to restart the collectd service when the service or plugins have been updated.

handlers/main.yml:

```bash
---
- name: restart collectd
  ansible.builtin.service:
    name: collectd
    state: restarted
```

- Enable the Write Prometheus plugin (use a Jinja2 template to create a config).

Create a Jinja2 template templates/prometheus.conf.j2:

```bash
LoadPlugin write_prometheus
<Plugin write_prometheus>
   Port 9103
</Plugin>
```

Add tasks to use the template in tasks/main.yml:

```bash
- name: Configure write_prometheus plugin
  ansible.builtin.template:
    src: prometheus.conf.j2
    dest: /etc/collectd/collectd.conf.d/prometheus.conf
  notify: restart collectd
```


Define a variable in defaults/main.yml to control the installation or removal of collectd:

```bash
---
collectd_state: present  # Use 'absent' to uninstall
```

2. Add to the collectd prometheus exporter config loading collectd modules:
- df
- processes
- protocols
- swap
- tcpconns
- uptime
- users
- vmem

Let's extend the prometheus.conf.j2 template to include the additional modules:

```bash
LoadPlugin write_prometheus
<Plugin write_prometheus>
   Port 9103
</Plugin>

# Additional modules
LoadPlugin df
LoadPlugin processes
LoadPlugin protocols
LoadPlugin swap
LoadPlugin tcpconns
LoadPlugin uptime
LoadPlugin users
LoadPlugin vmem
```

3. Add a new task to remove collectd service, collectd plugins, and collectd config files.

Add a conditional task in tasks/main.yml for removal if required:

```bash
- name: Set flag for collectd restart conditionally
  ansible.builtin.set_fact:
    restart_collectd: true
  when: collectd_state != 'absent'

- name: Install collectd
  ansible.builtin.package:
    name: collectd
    state: latest
  notify: restart collectd
  when: restart_collectd is defined and restart_collectd

- name: Configure write_prometheus plugin
  ansible.builtin.template:
    src: prometheus.conf.j2
    dest: /etc/collectd/collectd.conf.d/prometheus.conf
  notify: restart collectd
  when: restart_collectd is defined and restart_collectd

- name: Ensure collectd is enabled and started
  ansible.builtin.service:
    name: collectd
    state: started
    enabled: true
  when: collectd_state != 'absent'

- name: Stop and disable collectd service
  ansible.builtin.service:
    name: collectd
    state: stopped
    enabled: false
  when: collectd_state == 'absent'

- name: Remove collectd package
  ansible.builtin.package:
    name: collectd
    state: absent
  when: collectd_state == 'absent'

- name: Remove collectd configuration
  ansible.builtin.file:
    path: /etc/collectd.d/
    state: absent
  when: collectd_state == 'absent'
```

Create a playbook deploy_collectd.yml to apply the role:

```bash
---
- hosts: managed_nodes
  become: yes
  roles:
    - collectd
  vars:
    collectd_state: present  # Or 'absent' to remove
```

Execute the playbook with:

```bash
ansible-playbook -i /etc/ansible/hosts deploy_collectd.yml
```

![alt text](<screenshots/iScreen Shoter - Safari - 240221122423.png>)

![alt text](<screenshots/iScreen Shoter - Safari - 240221125534.png>)

The playbook successfully executed all tasks without any errors. The collectd service was installed, configured for the write_prometheus plugin, and ensured to be enabled and started across the managed nodes. The playbook also appropriately handled conditions for removal tasks, skipping them as designed.

Let's check the exported metrics running curl http://node1.example.com:9103 and curl http://node2.example.com:9103.

![alt text](<screenshots/iScreen Shoter - Safari - 240221125836.png>)

![alt text](<screenshots/iScreen Shoter - Safari - 240221130102.png>)

Now let's remove collectd service, collectd plugins, and collectd config files by setting collectd_state: absent in the ./collectd/defaults/main.yml file of the role and in the playbook.

![alt text](<screenshots/iScreen Shoter - Safari - 240221134858.png>)





