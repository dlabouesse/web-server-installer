# web-server-installer

Tool to install and configure web servers from scratch using Ansible.

## Setup

### Requirements

- You need a personal SSH key. The public key available in `~/.ssh/id_rsa.pub` will be attached to a new user created on your server(s).

### Configuration

1. Copy [hosts.yml.dist](hosts.yml.dist) into `hosts.yml`.
2. Copy [hosts.init.yml.dist](hosts.init.yml.dist) into `hosts.init.yml`.
3. For both `hosts.yml` and `hosts.init.yml` files:
    - Define your domains in the hosts list.
    - Replace the following variables:
        - `#user`: Desired username of the new user who will be attached to your SSH public key.
        - `#ssh_port`: Desired SSH port to use instead of the default 22 port.

## Usage

This script is made of the two following parts:

## 1. Init phase

The init phase:
- Creates the new user
- Sets the SSH key
- Gives sudo rights to this user
- Changes the SSH port
- Disables SSH password authentication
- Disables SSH root login

This phase connects to the server(s) as root and will prompt for the root password.

Run `ansible-playbook -i hosts.init.yml playbook.init.yml --ask-pass` to run this phase.

## 2. Install phase [WIP]

The install phase:
- Installs Docker

...

This phase connects to the server(s) as the specified user and uses your personal SSH key.

Run `ansible-playbook -i hosts.yml playbook.yml` to run this phase.

**You're all set!**
