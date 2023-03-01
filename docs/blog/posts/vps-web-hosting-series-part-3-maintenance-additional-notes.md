---
date: 2023-03-01
categories:
  - Bare metal
  - Hosting
  - Sysadmin
links:
  - "Part 1: Installation": blog/posts/vps-web-hosting-series-part-1-installation.md
  - "Part 2: Configuration": blog/posts/vps-web-hosting-series-part-2-configuration.md
---

![How to configure a VPS for web hosting?](/assets/images/blog/how-to-configure-a-vps-for-web-hosting/server.jpg)

# VPS & Web Hosting series part 3: Maintenance & additional notes

<!-- more -->

!!! warning

    This is the **third** part of a series:

    1. [VPS & Web Hosting series part 1: Installation](/blog/2023/02/27/vps-web-hosting-series-part-1-installation/)
    2. [VPS & Web Hosting series part 2: Configuration](/blog/2023/02/28/vps-web-hosting-series-part-2-configuration/)
    3. [VPS & Web Hosting series part 3: Maintenance & additional notes](/blog/2023/03/01/vps-web-hosting-series-part-3-maintenance-additional-notes/){ .current-page }

    If you didn't read the [VPS & Web Hosting series part 1: Installation](/blog/2023/02/27/vps-web-hosting-series-part-1-installation/)
    or [VPS & Web Hosting series part 2: Configuration](/blog/2023/02/28/vps-web-hosting-series-part-2-configuration/), please considere
    doing it before reading what follows :wink:

## Maintenance

The maintenance is minimal, since `certbot` auto renew all needed certificate automatically and reload `apache2` by its own.

We may want to login to reboot the server every month or so to apply some pending security updates.

The only other real maintenance you would want to do is to deploy a new itrocks version.

To deploy a new update, you must sync the server with the main repository `release` branch. So, first, you must ensure your changes have been pushed to the `release` branch, then all you have to do is a `git pull` on the `VPS`.

To do so, you have two choices :

1. Github action
2. Command line + `SSH`

### Github actions

The easy way.

Go to the main repository. In the `Actions` tab, selects `Deploy to VPS1 (production)`, then click the `run workflow` button, then the second `run workflow` button.

Wait for a minute or two for the process to end and... voilà. The server has been synchronized with the `release` branch.

### Command line + `SSH`

You will just do what the **Github Action** would have done, but by hand:

```bash
ssh ubuntu@156.225.137.143
cd ITRocks
rm -rf node_modules
git checkout release # to be sure we are on the right branch
git pull
npm install
sudo systemctl restart itrocks.target
```
{ no-linenums }

Check that both the target service and the itrocks instance are running:

```bash
sudo systemctl status itrocks.target
sudo systemctl status itrocks@1.service
```
{ no-linenums }

Then you can logout with `exit`.

## Additional notes

This section contains other configuration tips and reminders.

### Iptables (firewall)

If you add a new service that must be reached by the itrocks server or any software that you install on this server, **do not forget to check if a new rule must be added to iptables to open a new port**.

There is the exhaustive list of all open ports (**KEEP THIS LIST UP TO DATE !**):
 
- `22`    SSH
- `53`    apt-get
- `80`    HTTP
- `443`   HTTPS
- `465`   SSMTP
- `4430`  Websockets
- `27017` MongoDB

### Netdata

Netdata is a powerful monitoring tool that even comes with a panel. Fairly simple to install, just run:

```bash
wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh && sh /tmp/netdata-kickstart.sh
```
{ no-linenums }

By default, the panel is publicly available at `http://[server ip]:19999` but our firewall rules will hopefully block it. Since it's a useless config, we will use `apache2` to create a new proxy especially for `netdata`.

Let's start by issuing some self-signed certificates:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```
{ no-linenums }

Now we need to create a new `apache2` configuration:

```bash
sudo nano /etc/apache2/sites-available/vps1-ovh.netdata.itrocks.com.conf
```
{ no-linenums }

Paste this content into the editor:

```apache title="/etc/apache2/sites-available/vps1-ovh.netdata.itrocks.com.conf"
<VirtualHost *:443>
    ServerName vps1-ovh.netdata.itrocks.com

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

    ProxyRequests Off
    ProxyPreserveHost On

    <Proxy *>
        AllowOverride None
        AuthType Basic
        AuthName "Protected site"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Proxy>

    ProxyPass "/" "http://localhost:19999/" connectiontimeout=5 timeout=30 keepalive=on
    ProxyPassReverse "/" "http://localhost:19999/"

    ErrorLog ${APACHE_LOG_DIR}/netdata-error.log
    CustomLog ${APACHE_LOG_DIR}/netdata-access.log combined
</VirtualHost>
```

Then, type ++ctrl+s++ to save and ++ctrl+x++ to exit.

We will need to create a user and a password:

```bash
sudo htpasswd -c /etc/apache2/.htpasswd netdata
```
{ no-linenums }

We now need to enable this new website and reload `apache2`:

```bash
sudo a2ensite vps1-ovh.netdata.itrocks.com.conf
sudo systemctl reload apache2
```
{ no-linenums }

You can now browse [https://vps1-ovh.netdata.itrocks.com](https://vps1-ovh.netdata.itrocks.com) (once your domain have been setup to point to the server IP, obviously).

Accept the security exception regarding the self signed certificate, specify your username (in our case `netdata`) and the password you typed at the previous step and... voilà.

Metrics everywhere!


### Github automation

To allow for deployment directly from the main github repository in tab `Actions`, you must start by creating three secrets into `Settings` -> `Secrets`.

You must adapt their names to try to describe the new server instance with a meaningful name, **they must be different for each server**):

- `OVH_VPS1_HOST` : host ip or domain (for us it is `156.225.137.143`)
- `OVH_VPS1_USER` : main ssh user (for us it is `ubuntu`)
- `OVH_VPS1_PWD` : ssh pwd for the above user

Then you must create a new file in the `main` branch of the main repository in `.github/workflows`, let's call it `vps1.prod.yml` with the following content:

```yaml title=".github/workflows/vps1.prod.yml"
# This is a basic workflow that is manually triggered

name: Deploy to VPS1 (production)

on: workflow_dispatch

jobs:
  deploy_OVH_VPS1:
    runs-on: ubuntu-latest
    steps:
      - name: SSH Remote Commands
        uses: appleboy/ssh-action@v0.1.5
        with:
          host: ${{ secrets.OVH_VPS1_HOST }}
          username: ${{ secrets.OVH_VPS1_USER }}
          password: ${{ secrets.OVH_VPS1_PWD }}
          script: |
            cd ~/ITRocks
            rm -rf node_modules
            git checkout release
            git pull
            npm install
            sudo chown -R :itrocks node_modules
            sudo systemctl restart itrocks.target
            sudo systemctl status itrocks.target
```

**Think to replace the main name, and secrets names by the one you defined !**

!!! warning

    For the GithubAction to be able to run `npm install` with the right version of node,
    you **MUST** provide to the system a way to locate the good `node` binary to execute.

    To do so, please follow these few steps:
    
    1. Run the command `which node` to get the current used node path. There, it should be `/home/ubuntu/.nvm/versions/node/18.14.2/bin/node`
    2. Edit the file `/etc/envrionment` by running `nano /etc/environment`
    3. Add **the path** to the node binary by adding **at the beginning** of the `PATH` variable value `/home/ubuntu/.nvm/versions/node/v18.14.2/bin:`
    4. Type ++ctrl+s++ to save and ++ctrl+x++ to exit.

---------

Commit to the main branch, and voilà, you'll be able to [use your new GitHub Action as explained in the maintenace section](#github-actions), except you obviously want to select the action you just added.