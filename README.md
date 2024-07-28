# Ansible


Ansible is a configuration management tool whose main purpose is to help **automatically** provision servers

In a standard architecture, a machine (called an *ansible control host*) serves as a middle man between the operator of ansible (me) and the servers to provision. This control host connects directly to the servers (via SSH) and issues commands to provision them. 

An alternative to this, is to have the operator workstation (my laptop), act as a control host directly, skipping the machine in the middle. 

![](./img/ansible1.jpg)

> Its important to highlight that in both of these setups, the servers to be provisioned **don't** have Ansible installed


## Previous setup

### SSH

SSH needs to be configured, as it is the main way that ansible uses to send commands to the servers. This means that:

- The control node must have openssh client installed
- The servers must have openssh server installed

As general good practice, the use of ssh keys instead of passwords is recommended. There is multiple type of keys to generate, the example below generates a ED25519 key: 

```bash
ssh-keygen -t ed25519 -C "some key"
```

The key can have a password to unlock it. This is recommended for **personal** keys, but in the case of creating a key for ansible to use, it's better to skip it. 

Once a key is created it can be added to a server using the command:

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub <USER>@<IP>
```

For this the user must already exist in the machine, and the machine running the command should also have ssh access. 

> The first time that a connection to a new server is established, the ssh client will ask for confirmation. It is common to do this first connection manually to avoid "messing" with Ansible when it tries to connect. Alternatively, ansible can also be [configured to automatically accept](https://stackoverflow.com/questions/32297456/how-to-ignore-ansible-ssh-authenticity-checking)

### Git

In a normal production workflow, ansible scripts and playbooks are stored in a git repository. This allows to control and have a unified version of the scripts.


## Getting Started

### Installing Ansible

Ansible can be installed from their direct repository, using `pip` or directy from many linux distribution package stores. 

More info can be seen [here](https://docs.ansible.com/ansible-core/devel/installation_guide/installation_distros.html)

To install in ubuntu:

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

### Inventory File

To know which machines are available to control, ansible needs an inventory file. This file simply describes (with the ip addresses) which machines are to be controled

The inventory files allow also to group and set up different configuration values for each server. They can be defined in `ini` or `yaml` formats (depending what is easier to read given the architecture)

Even if no groups are defined in the inventory file, Ansible creates two default groups: `all` and `ungrouped`. The `all` group contains every host. The ungrouped group contains all hosts that don’t have another group aside from `all`. Threfore every host will always belong to at least 2 groups (`all` and `ungrouped` or `all` and some other group)

Once the inventory is organized, commands can be run on all host in a group with:
```bash
ansible mygroup ...
ansible all ...
```

More information can be found in the [documentation](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#how-to-build-your-inventory)



### Running ad-hoc commands

Although the core of ansible is using playbooks, the cli tool can be user to directly execute commands or run modules. They work great for tasks that are run not that ofter.  

For running a command the following is needed:
- Inventory group name 
- Key to use when connecting via SSH
- Inventory file location
- Module or command

This can all be passed in the cli. For example, the following command uses the `ping` [module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ping_module.html) to verify connection to all servers:

```bash
ansible all --key-file ansible.key -i inventory.ini -m ping
```

Most of the repetitive values however, can be put inside a `ansible.cfg` file. A local file will override configuration in `/etc/ansible/ansible.cfg`. [Configuration](https://docs.ansible.com/ansible/latest/reference_appendices/config.html) can also be done via environment variables:

```cfg
[defaults]
inventory = inventory.ini
private_key_file = ./ansible.key
remote_user = ansible
```

That way, the following command (that uses the `gather_facts` module to get information of the servers) works:

```bash
ansible all -m gather_facts
```

#### Elevated commands

Ansible uses existing privilege escalation systems to execute tasks with root privileges or with another user’s permissions. Because this feature allows you to ‘become’ another user, different from the user that logged into the machine (remote user), its called `become`. The `become` keyword uses existing privilege escalation tools like `sudo`, `su`, etc (using `sudo` by default)

If nothing else is specified, the `become` keyword will try to run the commands as the `root` user. The password can be passed via a file (using the `--become-password-file` option) or be prompted (`-K` or `--ask-become-pass`). More information can be seen in the [docs](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)

Example of ad-hoc elevated commands are:

- Update package cache in all servers (`apt update`)
    ```bash
    ansible all -m apt -a update_cache=true --become --ask-become-pass
    ```

- Update and upgrade all packages (`apt upgrade`)
    ```bash
    ansible all -m apt -a upgrade=yes --become --ask-become-pass
    ```

## Playbooks

Ansible Playbooks offer a repeatable, reusable, simple configuration management and multi-machine deployment system. They are files that allow to define a set of instructions to execute to reach a desired state. Sames as the ad-hoc commands, they make use of modules to execute things. 

They are written in YAML format and run with the `ansible-playbook` binary

```bash
ansible-playbook -K my_plabook.yml
```

An example of a simple playbook can be seen below:

```yaml
- name: Install apache # Name of the whole playbook
  hosts: all # Running on all nodes
  become: true # Same as --become, run this as root
  tasks:
  - name: install apache2 package # Name of the task
    apt: # Using the builtin apt module
      name: apache2 # Name of the package to install
      update_cache: true # Before installing run apt update
```

More modules, and options can be added to achieve complex behaviors.

### Running a playbook

When running a playbook, ansible first always runs a `gather_facts` task to gain information on the servers. After running, the results will be summarized in the `PLAY RECAP`, for example:

```bash
ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

This gives an overview of what the command achieved and wheter or not it succeded
- `ok` shows the tasks that did not fail (always 1 plus because of the `gather_facts` tasks)
- `changed` shows the tasks that actually changed something in the server (Running an install task twice will only change once)
- `unreachable` will report if the server could not be reached while running the task
- `skipped` will make sense for conditional tasks (that run depending if something is true)


## Conditionally running

### `when` keyword

Conditional logic can be introduced into Ansible playbooks using the `when` keyword. This allows to define a condition to execute a task. Any variable, including the ones resulted from `gather_facts` can be used for the condition. This means, that if a task should only run on Ubuntu servers, the task could look like this:

```yaml
  - name: install apache2 package # Name of the task
    apt: # Using the builtin apt module
      name: apache2 # Name of the package to install
      update_cache: true # Before installing run apt update
    when: ansible_distribution == "Ubuntu"
```

> `gather_facts` can be called as an ad-hoc task `ansible -m gather_facts` to see the different variables and values available


### `hosts` and groups

Another way to conditionally execute a playbook is to group the servers (based on characteristics, function, etc) and then using the `hosts` entry in the plays to target the different groups

```yaml
---
- name: Update packages
  hosts: all # Running on all nodes
  become: true 
  tasks:
  - name: install updates (centos)
    dnf:
      update_only: yes
      update_cache: yes
    when: ansible_ditribution == "CentOS"

- name: Provision web servers
  hosts: web_servers
  become: true
  tasks:
  - name: install apache2 package and php support (centos)
    dnf:
      name: 
        - httpd
        - php
      state: latest
    when: ansible_ditribution == "CentOS"

- name: Provision dbs
  hosts: db_servers
  become: true
  tasks:
    - name: install mariadb (centos)
      dnf:
        name: mariadb
        state: latest
```

With an inventory that looks like

```ini
[web_servers]
1.2.3.4

[db_servers]
5.6.7.8.9
```

### Tags

To focus (or skip) certain tasks from the playbook explicitly, tags can be used. This metadata allows ansible to just run or skip certain tasks. This helps to target particular architecture, quickly debug changes, etc. 

Tags are defined in the different tasks as a comma separated string

```yaml
- name: Update packages
  tags: always
  hosts: all # Running on all nodes
  become: true 
  tasks:
  - name: install updates (centos)
    dnf:
      update_only: yes
      update_cache: yes
    when: ansible_ditribution == "CentOS"

- name: Provision web servers
  tags: web_servers,centos
  hosts: web_servers
  become: true
  tasks:
  - name: install apache2 package and php support (centos)
    dnf:
      name: 
        - httpd
        - php
      state: latest
    when: ansible_ditribution == "CentOS"
```

Using the cli, `ansible-playbook --list-tags playbook.yml` can be used to see available tags. At the same time `ansible-playbook --tags centos` can be used to run only tasks that have a tag, with the opposite being `ansible-playbook --skip-tags centos`

The special keywords `always` and `never` can be used in the tags to guarantee that no matter which tags are passed, a task always (or never) runs


## Variables

Referenced in the playbook as `{{ name }}` variables allow for consolidation of playbooks as well as flexibility. 

Variables can be defined:
 
- In the inventory file. Particular values per host or group can be defined and will be picked up

```ini
[mygroup]
1.2.3.4 apache_package=apache2 

[mygroup:vars]
other_var=something
```


## Common use cases and modules

### Copying a file

- For this the `copy` module can be used
- For example:

```yaml
- name: Send custom html page
    copy:
      src: default_site.html # files directory is assumed
      dest: /var/www/html/index.html
      owner: root
      group: root
      mode: 0644
```

### Unzipping a file

- `unarchive` module can be used
- It supports downloading a file directly

```yaml
- name: Install terraform
  unarchive: # Needs unzip installed
    src: https://releases.hashicorp.com/terraform/0.12.28|terraform_0.12.28_linux_amd64.zip
    dest: /usr/local/bin
    remote_src: yes
    mode: 0755
    owner: root
    group: root
```

### Change lines in config files
- The `lineinfile` module allows to match a file and replace it in the target server
- It needs to be tested before hand, as any typo can lead to duplicate lines in the file leading to misconfiguration

```yaml
- name: change e-mail address for admin
     lineinfile:
       path: /etc/httpd/conf/httpd.conf
       regexp: '^ServerAdmin'
       line: ServerAdmin somebody@somewhere.net
     when: ansible_distribution == "CentOS"
     register: httpd
```

### Manage Services

- Services can be managed, started, restarted, enabled, etc. from ansible using the `service` module
- This is a generic module that acts  acts as a proxy to the underlying service manager module (like `systemd`) so not all options are available (similar to `package` module and `apt` or `dnf`)

```yaml
- name: Start and enable httpd in a CentOS machine
  service:
    name: httpd
    state: started
    enabled: true

- name: change e-mail address for admin
  lineinfile:
    path: /etc/httpd/conf/httpd.conf
    regexp: '^ServerAdmin'
    line: ServerAdmin somebody@somewhere.net
  when: ansible_distribution == "CentOS"
  register: httpd
 
- name: restart httpd (CentOS) only if the config file was changed
  service:
    name: httpd
    state: restarted
  when: httpd.changed 
```

### User management

- Several modules in ansible allow admins to do user management on the servers like `builtin.user` or `posix.authorized_key`
- A very common use case is to have a bootstrap playbook to provision a user which then ansible can use for other playbooks (by having passwordless sudo for example)

```yaml
- hosts: all
  become: true
  tasks:

  - name: create simone user
    user:
      name: simone
      groups: root
    
  - name: add ssh key for simone
    authorized_key:
      user: simone
      key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAe7/ofWLNBq3+fRn3UmgAizdicLs9vcS4Oj8VSOD1S/ ansible"
        
  - name: add sudoers file for simone
    copy:
      content: 'simone ALL=(ALL) NOPASSWD: ALL'
      dest: /etc/sudoers.d/simone
      owner: root
      group: root
      mode: 0440
```
