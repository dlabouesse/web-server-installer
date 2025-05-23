# web-server-installer

Tool to install and configure web servers from scratch using Ansible.

Tested with Debian 12 on OVHCloud VPS and KS3 server.

## Setup

### Requirements

- The server must have a user account with sudo rights. (Ensure the user account has a password as well as the root account.) To configure the passwords you can:
  - Run `sudo passwd {{user}}`
  - Run `sudo su -` then `passwd`
- Add your SSH public key to enable passwordless SSH login for the user account. (`ssh-copy-id -i ~/.ssh/id_rsa {{user}}@{{hostname}}`)
- On localhost, please install the required Ansible mobules `ansible-galaxy collection install -r requirements.yml`

### Configuration

1. Create a `hosts/{{domain}}` directory where `{{domain}}` is the domain name for the server to configure.
2. Copy [hosts.yml.dist](hosts.yml.dist) into `./hosts/{{domain}}/hosts.yml`.
3. Copy [hosts.init.yml.dist](hosts.init.yml.dist) into `hosts/{{domain}}/hosts.init.yml`.
4. For both `hosts.yml` and `hosts.init.yml` files:
    - Define your server IPs in the hosts list.
    - Replace the following variables:
        - `#hostname`: Desired domain name for the server to configure.
        - `#user`: username of the user account.
        - `#ssh_port`: Desired SSH port to use instead of the default 22 port.
        - `#admin_email`: Email address to receive admin notifications.

## Usage

This script is made of the two following parts:

### 1. Init phase

The init phase:
- Changes the SSH port
- Disables SSH password authentication
- Disables SSH root login

Run `ansible-playbook -i hosts/{{domain}}/hosts.init.yml playbook.init.yml` to run this phase.

### 2. Install phase

The install phase:
- Configures hostname
- Installs and configures Postfix
- Installs and configures OpenDKIM (Optional, see [Optional DKIM support](#Optional-DKIM-support))
- ~Optionally sets up weekly security updates~ (not tested on Debian 12)
- Installs and configures fail2ban. Also installs firewalld as firewall. If you experience with host being unreachable after reboot, you can defer firewalld startup by setting `defer_firewalld` to `true`.
- Installs PortSentry to prevent port scanning
- Installs Docker and Docker Compose
- Installs Caddy as a reverse proxy
- Installs and configure a monitoring interface
    - cAdvisor is installed to collect Docker containers monitoring data
    - Promotheus is installed to store cAdvisor data
    - Grafana is installed as monitoring dashboard with prepopulated items and is exposed through HTTPS using the `status` subdomain (defined in [roles/monitoring/defaults/main.yml](roles/monitoring/defaults/main.yml))
    - Optional DKIM support (see [Optional DKIM support](#Optional-DKIM-support))

This phase connects to the server(s) as the specified user and uses your personal SSH key.

Run `ansible-playbook -i hosts/{{domain}}/hosts.yml playbook.yml` to run this phase.

### 3. Post-install phase
- Update DKIM DNS records (if enabled, see [Optional DKIM support](#Optional-DKIM-support))
- Log in to Grafana with `admin/admin` credentials and change the password.
  - Test the SMTP configuration by sending a test email from the `Admin email` contact point.

**You're all set!**

---
**The following operations have not been tested on Debian 12.**

- Installs OpenVPN (optional)
    - Configures 2 OpenVPN servers with 2FA enabled
        - a first instance is configured on port 1194 using UDP
        - a fallback instance is configured on port 80 using TCP (Enables port-sharing to redirect usual HTTP requests to port 8080)
    - Generates client certificate(s) and downloads the configuration file(s) (See [Install an additional VPN client certificate](#install-an-additional-vpn-client-certificate))
    - Generates QRCode URLs to configure 2FA for each client
- Installs Samba (optional)
    - Configure one or multiple shares
    - Exposes a Samba server into a Docker container with a fixed IP address (172.21.0.4 by default), only reachable from the VPN subnet
- Installs Nextcloud (optional)
    - Creates admin user account
    - Preconfigures SMTP server
    - See below for [additional manual operations](#nextcloud-manual-operations)
- Installs WordPress website(s) (optional) (See [Install an additional WordPress website](#install-an-additional-WordPress-website))
- Installs PrestaShop website(s) (optional) (See [Install an additional PrestaShop website](#Install-an-additional-PrestaShop-website))
---

## Version control

According to [documentation](https://grafana.com/docs/loki/latest/installation/docker/#install-with-docker), Loki and Promtail versions are fixed to 3.2.1 in [roles/monitoring/templates/compose.yml.j2](roles/monitoring/templates/compose.yml.j2) and [roles/monitoring/tasks/main.yml](roles/monitoring/tasks/main.yml).

## Optional DKIM support

DKIM support can be enabled by defining the `enable_dkim` variable to `true` in `hosts.yml`.

To ensure deliverability of emails sent to the same domain than the server hostname, enabling DKIM requires a [hubbed_hosts](https://manpages.debian.org/jessie/exim4-config/exim4-config_files.5.en.html#/etc/exim4/hubbed_hosts) file located at `hosts/{{domain}}/hubbed_hosts`. If no mails are sent to the server hostname, you can leave this file empty.

This script creates two separate DKIM configurations:
- The first one uses the `server` selector, and is used for system emails like cron tasks.
- The second one uses the `status` selector, and is used for grafana alerts.

The scripts generate the RSA keys on the server side, and retrieves the details of the DNS records to configure in a `{{selector}}._domainkey.txt` file located in the `hosts/{{domain}}/` directory, where `{{selector}}` corresponds to the respective DKIM selector.

## Auto updates
Auto updates for reverse proxy and monitoring containers can be enabled be defining the `auto_update` variable to `true` in `hosts.yml`.

If auto updates are disabled, these containers can be updated manually by running `ansible-playbook -i hosts/{{domain}}/hosts.yml playbook.yml --tags=caddy --tags=monitoring`.

---
**The following features have not been tested with Debian 11**

## NextCloud manual operations

NextCloud install requires few additional operations to be fully optimized.

Go to /settings/admin/overview, and if it is suggested to run `occ db:add-missing-indices` and `occ db:convert-filecache-bigint` then run:
- `docker-compose exec -u www-data nextcloud php occ db:add-missing-indices`
- `docker-compose exec -u www-data nextcloud php occ db:convert-filecache-bigint`

## Install an additional VPN client certificate

In `hosts.yml`, add an item in the `vpn_client_certificates` list, and set:
- `name`
- `passphrase`

Then, run `ansible-playbook -i hosts/{{domain}}/hosts.yml playbook.yml --tags=openvpn` to create the new certificate.

## Install an additional WordPress website

In `hosts.yml`, add an item in the `wordpress_websites` list, and set:
- `hostname` (The website will be installed in `/var/www/hostname/`)
- `name` (Impacts the composer project name and the db name)
- `db_user`
- `db_password`
- `db_root_password`

Then, run `ansible-playbook -i hosts/{{domain}}/hosts.yml playbook.yml --tags=wordpress` to deploy the new website.

## Install an additional PrestaShop website

In `hosts.yml`, add an item in the `prestashop_websites` list, and set:
- `hostname` (The website will be installed in `/var/www/hostname/`)
- `name` (Impacts the composer project name and the db name)
- `db_user`
- `db_password`
- `db_root_password`
- `admin_password` (Used to access the back office. The login is the value of `admin_email`)

Then, run `ansible-playbook -i hosts/{{domain}}/hosts.yml playbook.yml --tags=prestashop` to deploy the new website.
