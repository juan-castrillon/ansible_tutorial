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