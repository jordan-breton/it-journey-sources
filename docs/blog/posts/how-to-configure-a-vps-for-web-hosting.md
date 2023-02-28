---
date: 2023-02-27
categories:
  - Bare metal
  - Hosting
  - Sysadmin
---

![How to configure a VPS for web hosting?](/assets/images/blog/how-to-configure-a-vps-for-web-hosting/server.jpg)

# How to configure a VPS for web hosting?

Fully configuring a VPS is not that hard nor that long, but when we start our IT journey it's an intimidating task.

Today, we'll configure one from scratch (and for fun :wink:). Lets dream about a **NodeJS** application that must connect
to a **MongoDB** instance hosted somewhere else. To spice-it up a little, our fictive project called **IT Rocks** is both
an **HTTP** server and a **WebSocket** server. It must be able to send mails to our customers too.

And... since it's a very serious project, it needs to be accessible within restrictive networks as well. Why? Because... what would be 
an IT task without a tricky part?

<!-- more -->

!!! warning "Important"

    A few precisions before we start:
    
    1. This is inspired by several real-world scenarios I encountered in the past few years
    2. This tutorial is an **advanced** one.
    3. I tried my best to write it in a way that **should** allow a beginner to not be lost and reproduce every step without struggling too much
    4. This is not only once of those **How to?** article you may be used to read, it's also a **Why?** article. It means that I tried to justify why some 
       decisions have been taken for you to understand that tools and choices are not default ones. They have been thought to fit the project's requirements.
    
    As a consequence, despite my efforts to make it as short and digest as possible, it is quite long and detailed. But so are all real-case IT scenarios we encounter in our jobs.

    If you're a beginner, you should find here some valuable informations and resources that are not that easy to find and compile by yourself. But it comes at a cost:
    you'll have to invest some time and efforts to understand everything. If something is unclear or if you have a question... just leave a comment :)
    
## Project precisions & Preparation

First of all, we need a [**VPS** (Virtual Private Server) up and running](/blog){ target="_blank" } to connect to.

To sum up the needs defined in the intro and for clarity sakes, there are the requirements:
{ id="requirements-list" }

- NodeJS application
- HTTP server with [express](https://www.npmjs.com/package/express){ target="_blank" }
- WebSocket server with [uWebSocket.js](https://github.com/uNetworking/uWebSockets.js/){ target="_blank" }
- Sends mails
- Connects to distant **MongoDB** database service.
- Accessible in **restrictive networks** (entreprise, schools, etc.)
- The project sources are hosted in a Github **private** repository

??? note "About *express* and *uWebSocket.js*"

    This is not as uncommon as it seems to encounter a scenario like this. You can read more about *express* and *uWebSocket.js*
    configuration pitfalls [there](/writing), but to not make this tutorial too long, I'll sum it up in one sentance:

    **Express** and **uWebSocket.js** are **incompatible**. It means that **ITRocks** **needs** two open ports. It's important
    because of the *"accesible in restrctive networks"* requirement.

Now, it's time to roll up our sleeves and start typing on our beautiful keyboard :wink:

## Installation

The first thing to do is obviously to connect to our VPS to install the tools we need.

We ordered our **VPS** few minutes ago, and since our provider is as fast and efficient as our beautiful application,
we already received a mail with the following information:

<div class="annotate" markdown>

> Dear customer,
>
> `Ubuntu 20.04` operating system have been installed on the VPS you ordered:
>
> - IPv4 address: `156.225.137.143` (1)
> - VPS name: `ac78ef7.vps.provider.com` (2)
>
> The admin account configured on your VPS is:
>
> - User: `ubuntu` (3)
> - Password: `Ohb78@hioude!ma09k` (4)

</div>

1.  This is a **fictive** IPv4
2.  This is a **fictive** domain
3.  This is a **fictive** username
4.  This is a **fictive** password

!!! danger "Please, ^^DO NOT^^ use the above credentials as is. ^^CHANGE THEM^^!"

!!! note "The following tutorial should work for Ubuntu Server 22.04 as well. If you use another distribution, the following tutorial may not be accurate nor relevant anymore."

### First connection and ubuntu upgrade

Let's connect to the server for the first time through **SSH** with the command `ssh [user]@[ip]` or `ssh [user]@[domain]`. For the current VPS,
the exact command using the IP address is:

```bash
ssh ubuntu@156.225.137.143
```
{ no-linenums }

!!! note "The first time, it will print you a warning because of self-signed certificate, just accept it."

Then type the server password.

### Securing SSH connection to the VPS

Once logged in, we **must** start by **changing the admin password**, since it have been provided in cleartext by mail,
its security maybe compromised already, or maybe compromised later.

```bash
sudo passwd ubuntu
```
{ no-linenums }

Now, the first thing we want to do is disabling **SSH** password login to prefer a **private key** secured by **a passphrase**.
This reduces risks of successfully bruteforce attack to 0. And your SSH login should remain secure as long as the machine you keep
this private key on stays secure too.

!!! warning "For the whole tutorial, I'll assume that you're operating form an ubuntu local machine. If it is not the case, please adapt local commands to your operating system."

```bash title="On local machine"
# generate a new key pair (1)
ssh-keygen -t ed25519

# install the public key on the remote server
# Will ask the password we redefined above
ssh-copy-id -i ${HOME}/.ssh/id_ed25519.pub ubuntu@156.225.137.143
```
{ no-linenums }

1.  At this step, you will be given the option to setup a passphrase or not. Keep the habit to **always** define a passphrase. It makes your private key much harder to steal.

Then, try to connect to your **VPS** with the command `ssh ubuntu@156.225.137.143`. Your **operating system** will ask you the
**passphrase** you chose while generating the certificates, but if the operation was successful, the **SSH** client will not ask you the `ubuntu`
password.

Now, we want to prevent anyone to access the server with any password, to enforce keys usage, as well as forbid entirely root login. 
Lets create a new ssh config file:

```bash
sudo nano /etc/ssh/sshd_config.d/disable-password-authentication.conf
```
{ no-linenums }

With the following content:

```
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
PermitRootLogin no
```
{ no-linenums }

Type ++ctrl+s++ to save and ++ctrl+x++ to exit.

Then, we need to reload the SSH server and the SSH daemon:

```bash
sudo systemctl reload ssh
sudo systemctl reload sshd
```
{ no-linenums }

Test time. Trying to log with root user should be always rejected, so:

```bash
ssh root@156.225.137.143
```
{ no-linenums }

Must print `Permission denied (publickey).`, and:

```bash
ssh ubuntu@156.225.137.143 -o PubkeyAuthentication=no #(1)!
```
{ no-linenums }

1.  This option allow us to disable automatic key based authentication to request a password authentication

Must print `Permission denied (publickey).` too.

### Upgrading ubuntu & setting up automatic upgrades

Now that we successfully secured **SSH** access to our **VPS**, lets login again to upgrade *ubuntu*:

```bash
ssh ubuntu@156.225.137.143
sudo apt-get update
sudo apt-get upgrade
sudo reboot
``` 
{ no-linenums }

!!! note "Rebooting may or may not be mandatory, so we just go with it anyway."

Since we don't want to connect every day just to update the server to get security fixes, we will use 
the `unattended-upgrades` package that will take care of that for us:

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install unattended-upgrades
```
{ no-linenums }

Then we must enable it:

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```
{ no-linenums }

Just choose `yes` and press enter.

We **do not** turn on automatic reboot, because we may want to reboot at a specific point in time. ITRocks may want
to schedule a maintenance downtime, for example.

### Installing needed packages

Let's install what we need to run **IT Rocks** behind an apache proxy:

```bash
sudo apt-get install -y apache2 npm certbot python3-certbot-apache fail2ban iptables-persistent
```
{ no-linenums }

!!! note "The package iptables-persistent may ask you if you want to save the current `ipv4` or `ìpv6` config. The answer doesn't matter, since we will redefine them later anyway."

??? question "Wait... what are all those packages?"

    It looks like we're installing a bunch of software, let's explain what is what:
    
    | package                  | description                                                                                                   |
    |--------------------------|---------------------------------------------------------------------------------------------------------------|
    | `apache2`                | An HTTP server and will be our proxy. It could have been `NGinx` or `Caddy` too for example.                  |
    | `npm`                    | The NodeJS Package Manager, needed to fetch our project's dependencies.                                       |
    | `certbot`                | An utility that is useful to manage (generate/install/renew/revoke) SSL certificates signed by Let's Encrypt. |
    | `python3-certbot-apache` | A certbot plugin to work with apache. It will automatically setup apache configuration for us in a blink.     |
    | `fail2ban`               | A log analyzer that work with the firewall to ban suspicious activities.                                      |
    | `iptables-persistent`    | An utility that will persist our firewall configuration.                                                      |

??? question "Why do we need a proxy?"

    It seems that the time has come to discuss what we'll install on this server to fulfill our [requirements list](#requirements-list).
    The most important information in this requirements list now is the need for **2 open ports** ^^and^^ the need for access in **most restrictive networks**.
    But, what actually are restrictive networks? Why is that important?
    
    If at home your gateway (the device that connects your home network to Internet) do not restrict your online activities,
    in an entreprise or school network the gateway firewall will most likely block the inbound and outbound trafic on almost every
    ports. Not so long ago, it left two ports open to internet: 
    
    - 80 (HTTP)
    - 443 (HTTPS)
    
    Since port 80 is **insecure** and **SSL** certificates free for years now thanks to our savior [Let's Encrypt](https://letsencrypt.org/),
    port **80** almost disappeared from most of the firewalls' configurations. As such, it leaves us with... port **443**. However, our wonderful
    application still needs two open ports.
    
    It looks like we're screwed, isnt'it?
    
    Not even close. God bless **proxies**!
    
    If you don't know what a proxy is: it's something that acts like an interface between two other things. When it receives a request,
    it will not answer it himself but rather will forward the request to another service to get the answer, and then send it back to the requester.
    
    So we can setup a proxy that will expose only one **public port** (443 in our case), and route the trafic on two **private ports** on which 
    our **ITRocks** server will be listening. Another welcome benefit is that you can reload the proxy when SSL certificates are renewed without
    closing ongoing connections. Pretty cool, huh?

### NVM & node

Then we'll need **nvm** to manage **NodeJS** versions on the same machine, it will ease switching version later if needed:

```bash
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
```
{ no-linenums }

!!! note "This command downloads the last `nvm` version **at the moment this document has been written**, to install the last version, please read the [install section of their github](https://github.com/nvm-sh/nvm#install--update-script){ target="_blank" }"

Then we install the version of node that will run our server. `Hydrogen` is the last **LTS** (v18.14.2) version at the moment this document have been written:

```bash
nvm install Hydrogen
```
{ no-linenums }

While writing this doc, **Hydrogen** downloads the **v18.14.2**.

!!! note "You can use `nvm ls-remote` to print all available versions of node, and use `nvm ls-remote | grep LTS` to filter LTS versions only."

It will be set to `default`, you can check node version with `node -v`. If it doesn't print the version you installed, run the commands:

```bash
nvm alias default v18.14.2
nvm use v18.14.2
```
{ no-linenums }

Then, `node -v` should print `v18.14.2`

!!! warning "Very important note"

    If a **Github Action** is configured for the current server, you must follow [additional instructions](#github-automation)
    for this version change to take effect when the **Github Action** will run `npm install` !

    For now, it shouldn't. But if you come back later or if you didn't followed all steps and encounter strange error messages
    while trying to run a **Github Action**, carefuly read the above link.

### Installing ITRocks

We start by creating a folder in ubuntu's home folder `/home/ubuntu`:

```bash
cd ~
mkdir ITRocks
cd ITRocks
git init
git remote add origin git@github.com:example/ITRocks.git
git config core.fileMode false
```
{ no-linenums }

??? question "`git config core.fileMode false`, is this git sorcery?" 

    The command `git config core.fileMode false` is mandatory to **avoid** git to track **local file system permissions changes**. 
    
    If you don't run this command, when the git hooks that we will setup below will change group ownership of all files, 
    git will later prevent to checkout/pull before stashing or committing.

Now, we need to create a `deploy token` to be able to pull, since the repository is private (do not change generated file path/name, you don't need a passphrase):

```bash
ssh-keygen -t rsa -b 4096 -C example@itrocks.com
```
{ no-linenums }

Then we will configure the key for git:

```bash
cd ../.ssh
mv id_rsa ITRocks.deploy.pem
chmod 400 ITRocks.deploy.pem
nano config
```
{ no-linenums }

Type this into the file editor:

``` title="/home/ubuntu/.ssh/config"
Host ITRocks
Hostname github.com
IdentityFile ~/.ssh/ITRocks.deploy.pem
```

Then type ++ctrl+s++ to save and ++ctrl+x++ to exit.

We want to adapt permissions to this file as well:

```bash
chmod 770 config
```
{ no-linenums }

Now, you need to copy the content of `id_rsa.pub`. You can use the linux `scp` command from another terminal to download it, or just print it to the console to be able to copy it into your clipboard:

```bash
cat id_rsa.pub
``` 
{ no-linenums }

The public key must be set as `deploy key` on `GitHub`. To do so, go the the project repository, click on the `settings` in the top tabs, then choose `Deploy Keys` in the left panel, and click the `Add deploy key` button, top right.

Type a meaningful name as `Public production key on [Provider] VPS` as title, and paste the previously copied `ìd_rsa.pub` file content into the `key` field. **Do not check the `Allow write access` checkbox** for security sakes.

Once done, come back to your **SSH** terminal and type :

```bash
cd ~/ITRocks
git config core.sshCommand 'ssh -i /home/ubuntu/.ssh/ITRocks.deploy.pem'
git pull
git checkout release
```
{ no-linenums }

We're now ready for the configuration process.

## Configuration

In this section we will configure `apache2`, `certbot`, `systemctl`, `logrotate`, `iptables` and `itrocks` to get a fully operational production server. If it has not been done by then, read carefully the [installation section](#installation) before trying to follow below instructions.

For security sakes, we will create a specific user and a specific group to restrain our server ability to access the file system:

```bash
sudo addgroup itrocks
sudo useradd itrocks -g itrocks
```
{ no-linenums }

Some permissions must be setup for `itrocks` to be able to access the folder and run node:

```bash
cd ~
chmod o+x .
chmod o+x -R .nvm
```
{ no-linenums }

Now, we need to add our main user `ubuntu` to the newly created group `itrocks`:

```bash
sudo usermod -aG itrocks ubuntu
```
{ no-linenums }

!!! warning "Before continuing, you ^^MUST^^ run the `exit` command and reconnect through SSH for this change to take effects."

For git to automatically setup right permissions/ownership to all files when we pull/checkout, let's create a script in `/home/ubuntu/tools`:

```bash
cd ~
mkdir tools
nano tools/gitHook.sh
```
{ no-linenums }

Then, past this content to the editor:

```bash title="/home/ubuntu/tools/gitHook.sh"
#!/bin/bash

PATH="/home/ubuntu/ITRocks"
/bin/chmod -R 771 "$PATH"
/bin/chown -R :itrocks "$PATH"
```

Type ++ctrl+s++ to save and ++ctrl+x++ to exit.

Then give the execution permission to the script:

```bash
chmod +x tools/gitHook.sh
```
{ no-linenums }

Now, we will create two hooks in `/home/ubuntu/ITRocks/.git/hooks` by symlinking our script:

```bash
ln -s /home/ubuntu/tools/gitHook.sh /home/ubuntu/ITRocks/.git/hooks/post-checkout
ln -s /home/ubuntu/tools/gitHook.sh /home/ubuntu/ITRocks/.git/hooks/post-merge
```
{ no-linenums }

Test it by checkout to the master branch and checkout back to the release branch, then use `ls` to check all files and folders belongs to the `itrocks` group:

```bash
cd ITRocks
git checkout master
git checkout release
ls -al
```
{ no-linenums }

The `ls -al` command should print something like this:

```
drwxrwx---  8 ubuntu itrocks   4096 Oct 12 17:42 .
drwxr-x---  9 ubuntu ubuntu    4096 Oct 13 07:20 ..
drwxrwx---  8 ubuntu itrocks   4096 Oct 13 07:21 .git
-rwxrwx---  1 ubuntu itrocks     77 Oct 12 17:41 .gitignore
-rwxrwx---  1 ubuntu itrocks   1814 Oct 12 17:42 package.json
-rwxrwx---  1 ubuntu itrocks 175160 Oct 12 17:42 package-lock.json
-rwxrwx---  1 ubuntu itrocks   3896 Oct 12 17:45 index.js
```
{ no-linenums }

### Apache2

Let's start by enabling required modules:

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel ssl
```
{ no-linenums }

Then, we will create the file `/etc/apache2/sites-available/app.itrocks.com.conf` :

```apache
<VirtualHost *:80>
        ServerName app.itrocks.com

        ProxyRequests Off
        ProxyPreserveHost On
        ProxyVia Full
        <Proxy *>
                Require all granted
        </Proxy>

        ProxyPass / http://127.0.0.1:7000/
        ProxyPassReverse / http://127.0.0.1:7000/
</VirtualHost>
```

!!! important "For now, SSL related config is ignored, certbot will edit this file for us. We just need to provide him a working config on an insecure VirtualHost."

Now we want to enable our website:

```bash
sudo a2ensite app.itrocks.com.conf
sudo systemctl reload apache2
```

But before running certbot, we need a running server.


### ITRocks

For now, the `ITRocks/server/.env.production` file is shipped with the code. So... there is nothing to do.

Just check that, according to the previous installation process and `apache2` config, you run the server with the following config values:

- `SERVER_MODE=insecure` because we configure itrocks behind a proxy
- `PORT=7000` private ports on 127.0.0.1
- `WS_PORT=7001` private port on 127.0.0.1
- `WS_SERVER=uWebSocket`
- `WS_PROXY=1`
- `WS_PROXY_PORT=4430` publicly exposed port for websockets
- `INSTANCE=PROD_OVH_VPS1` to properly identify any instance in automatically reported errors.

### Systemctl

This is a service manager used to manage the lifecycle of our `itrocks` instance. We will create two files in `/etc/systemd/system/`:

- `itrocks@.service`: a templated unit that will allow us, later, to run multiple instances of itrocks on the same server. When horizontal scaling will be supported.
- `itrocks.target`: the service unit that will decide how much instances to run at a time and manage them for us. Systemd will ensure itrocks is running from server startup to server shutdown and relaunch it if it crashes.

Let's start by `/etc/systemd/system/itrocks@.service`.

We will need to know the path to the node binaries:

```bash
which node
```

Should print something like `/home/ubuntu/.nvm/versions/node/v16.17.1/bin/node`. We will use it to define the first systemctl unit in `ExecStart`. So, do not forget to replace this path if it is different in your environment.

Then create the file by typing `sudo nano /etc/systemd/system/itrocks@.service` and pasting the following content into the editor:

```systemd
[Unit]
Description = "ITRocks instance %i"
Requires = network.target
After = network.target
PartOf = itrocks.target

[Service]
User = itrocks
Restart = always
WorkingDirectory = /home/ubuntu/ITRocks
ExecStart = env NODE_ENV=production /home/ubuntu/.nvm/versions/node/v16.17.1/bin/node ITRocks/server/index
StandardOutput = append:/var/log/itrocks.%i.instance.log
StandardError = append:/var/log/itrocks.err.%i.instance.log

[Install]
WantedBy = multi-user.target
```

Type ++ctrl+s++ to save and ++ctrl+x++ to exit.

Then we will create `/etc/systemd/system/itrocks.target` by typing `sudo nano /etc/systemd/system/itrocks.target`:

```systemd
[Unit]
Description=ITRocks instance(s)
# To add more instances, just add a new service in the list by increasing the service number:
# Requires=itrocks@1.service itrocks@2.service picakform@3.service ...
Requires=itrocks@1.service

[Install]
WantedBy=multi-user.target
```

Again, type ++ctrl+s++ to save and ++ctrl+x++ to exit.

Our services are ready, we just need to activate them:

```bash
sudo systemctl enable itrocks.target
sudo systemctl start itrocks.target
```

To manage ITRocks now, you can:

- run `sudo systemctl start itrocks.target`
- run `sudo systemctl status itrocks.target`
- run `sudo systemctl stop itrocks.target`
- run `sudo systemctl restart itrocks.target`

But you can also use those commands to manage each instance individually (if there is several instances in the future):

- run `sudo systemctl start itrocks@1`
- run `sudo systemctl status itrocks@1`
- run `sudo systemctl stop itrocks@1`
- run `sudo systemctl restart itrocks@1`

### Logrotate

Logrotate will rotate our logs to compress them and remove them automatically after some delay.

To configure it, just run:

```bash
nano /etc/logrotate.d/itrocks
```

And paste the following content in the editor:

```
/var/log/itrocks.*.log {
       daily
       rotate 14
       delaycompress
       compress
       notifempty
       missingok
}
```

Then, type ++ctrl+s++ to save and ++ctrl+x++ to exit.

This configuration will rotate itrocks logs daily and keep them 14 days before cleaning them up.

### Certbot

Now, we want to enable **SSL** for our proxy.

Just run:

```bash
sudo certbot --apache
```

Type the maintainer's mail address, select all domains and accepts terms and conditions.

Once certbot executed, we must edit the SSL config to ensure our websockets will be proxied too.

In our case, the apache file was `/etc/apache2/sites-available/app.itrocks.com.conf`, so certbot will generate a file named `/etc/apache2/sites-available/app.itrocks.com.conf-le-ssl.conf`:

```
sudo nano /etc/apache2/sites-available/app.itrocks.com.conf-le-ssl.conf
```

You should see something like this:

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
        ServerName app.itrocks.com
        ServerAlias apps.itrocks.com
        ServerAlias app.itrocks.fr
        ServerAlias apps.itrocks.fr

        DocumentRoot /var/www/html

        ProxyRequests Off
        ProxyPreserveHost On
        ProxyVia Full
        <Proxy *>
                Require all granted
        </Proxy>

        RewriteEngine On
        RewriteCond %{HTTP:Upgrade} =websocket [NC]
        RewriteRule /(.*)           ws://127.0.0.1:7000/$1 [P,L]
        RewriteCond %{HTTP:Upgrade} !=websocket [NC]
        RewriteRule /(.*)           http://127.0.0.1:7000/$1 [P,L]

        ProxyPassReverse / http://127.0.0.1:7000/

        Include /etc/letsencrypt/options-ssl-apache.conf
        SSLCertificateFile /etc/letsencrypt/live/app.itrocks.com/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/app.itrocks.com/privkey.pem
</VirtualHost>
</IfModule>
```

We need to add our websocket configuration just after the first `VirtualHost`:

```apache
Listen 4430
<VirtualHost *:4430>
        ServerName app.itrocks.com
        ServerAlias apps.itrocks.com
        ServerAlias app.itrocks.fr
        ServerAlias apps.itrocks.fr

        DocumentRoot /var/www/html

        ProxyRequests Off
        ProxyPreserveHost On
        ProxyVia Full
        <Proxy *>
                Require all granted
        </Proxy>

        ProxyPass / ws://127.0.0.1:7001 retry=0
        ProxyPassReverse / ws://127.0.0.1:7001

        Include /etc/letsencrypt/options-ssl-apache.conf
        SSLCertificateFile /etc/letsencrypt/live/app.itrocks.com/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/app.itrocks.com/privkey.pem
</VirtualHost>
```

The final file should be:

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
        ServerName app.itrocks.com
        ServerAlias apps.itrocks.com
        ServerAlias app.itrocks.fr
        ServerAlias apps.itrocks.fr

        DocumentRoot /var/www/html

        ProxyRequests Off
        ProxyPreserveHost On
        ProxyVia Full
        <Proxy *>
                Require all granted
        </Proxy>

        ProxyPass / http://127.0.0.1:7000/
        ProxyPassReverse / http://127.0.0.1:7000/

        Include /etc/letsencrypt/options-ssl-apache.conf
        SSLCertificateFile /etc/letsencrypt/live/app.itrocks.com/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/app.itrocks.com/privkey.pem
</VirtualHost>
Listen 4430
<VirtualHost *:4430>
        ServerName app.itrocks.com
        ServerAlias apps.itrocks.com
        ServerAlias app.itrocks.fr
        ServerAlias apps.itrocks.fr

        DocumentRoot /var/www/html

        ProxyRequests Off
        ProxyPreserveHost On
        ProxyVia Full
        <Proxy *>
                Require all granted
        </Proxy>

        ProxyPass / ws://127.0.0.1:7001 retry=0
        ProxyPassReverse / ws://127.0.0.1:7001

        Include /etc/letsencrypt/options-ssl-apache.conf
        SSLCertificateFile /etc/letsencrypt/live/app.itrocks.com/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/app.itrocks.com/privkey.pem
</VirtualHost>
</IfModule>
```

Then, type ++ctrl+s++ to save and ++ctrl+x++ to exit.

```bash
sudo systemctl reload apache2
```

And this is the end of it! ITRocks is now reachable on four urls:

- https://app.itrocks.com/client/itrocks/index.html#ui=authentication-login
- https://apps.itrocks.com/client/itrocks/index.html#ui=authentication-login
- https://app.itrocks.fr/client/itrocks/index.html#ui=authentication-login
- https://apps.itrocks.fr/client/itrocks/index.html#ui=authentication-login

### Iptables

Now that we have a running server, let's secure it with ubuntu's `iptables` firewall.

We will create a file in `/home/ubuntu/tools` called `ìptables.config.sh`:

```bash
cd ~/tools
nano iptables.config.sh
``` 

Past this content into the editor:

```sh
#!/bin/bash

# Empty current tables
iptables -t filter -F

# Empty personal rules
iptables -t filter -X

# Deny all
iptables -t filter -P INPUT DROP
iptables -t filter -P FORWARD DROP
iptables -t filter -P OUTPUT DROP

# Do not break established connections
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

# Allow loopback
iptables -t filter -A INPUT -i lo -j ACCEPT
iptables -t filter -A OUTPUT -o lo -j ACCEPT

# ICMP (ping)
iptables -t filter -A INPUT -p icmp -j ACCEPT
iptables -t filter -A OUTPUT -p icmp -j ACCEPT

# SSH 
iptables -t filter -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -t filter -A OUTPUT -p tcp --dport 22 -j ACCEPT

# DNS (bind) needed by apt-get to update packages
iptables -t filter -A OUTPUT -p tcp --dport 53 -j ACCEPT
iptables -t filter -A OUTPUT -p udp --dport 53 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 53 -j ACCEPT
iptables -t filter -A INPUT -p udp --dport 53 -j ACCEPT

# APACHE : HTTP + HTTPS
iptables -t filter -A OUTPUT -p tcp --dport 80 -j ACCEPT
iptables -t filter -A OUTPUT -p tcp --dport 443 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 443 -j ACCEPT

# ITRocks websockets through apache's proxy
iptables -t filter -A OUTPUT -p tcp --dport 4430 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 4430 -j ACCEPT

# Allow distant mongodb connection
iptables -t filter -A OUTPUT -p tcp --dport 27017 -j ACCEPT
iptables -t filter -A INPUT  -p tcp --dport 27017 -j ACCEPT

# SMTP Port
iptables -t filter -A OUTPUT -p tcp --dport 465 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 465 -j ACCEPT
```

**Beware:** Modifying this file may lock you out the server. This is still temporary at this step, because those changes only happen in memory. If you mess it up, restart the server by using your provider's admin panel. If restart doesn't works because the restart command sent by your provider use a blocked port, you'll need to rely on your provider support or on any recovery procedure he planned.

Then, type ++ctrl+s++ to save and ++ctrl+x++ to exit.

Give execution permission to this newly created file and run it to apply:

```bash
chmod +x iptables.config.sh
sudo ./iptables.config.sh
```

**Before persisting those rules in the next step, it is very important to test that everything is working fine:**

1) Exit and reconnect in `SSH` to ensure it is still possible
2) Try to reach itrocks and make it send a mail to one of your addresses (ensure that you receive it)
3) Login to itrocks and check that your websocket is successfully connected.
4) Try to run `sudo apt-get update` and `sudo apt-get upgrade` to ensure that the firewall rules do not prevent those commands to execute.

Once all is ok, we can persist iptables configuration:

```bash
sudo iptables-save > rules.v4
sudo mv rules.v4 /etc/iptables
```

Check that  `iptables-persistent` is running:

```bash
sudo systemctl status netfilter-persistent.service
```

Should print something like:

```
● netfilter-persistent.service - netfilter persistent configuration
     Loaded: loaded (/lib/systemd/system/netfilter-persistent.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/netfilter-persistent.service.d
             └─iptables.conf
     Active: active (exited) since Thu 2022-10-13 23:43:04 UTC; 19h ago
       Docs: man:netfilter-persistent(8)
   Main PID: 518 (code=exited, status=0/SUCCESS)
        CPU: 7ms

Oct 13 23:43:04 vps-6e61d811 systemd[1]: Starting netfilter persistent configuration...
Oct 13 23:43:04 vps-6e61d811 netfilter-persistent[522]: run-parts: executing /usr/share/netfilter-persistent/plugin>
Oct 13 23:43:04 vps-6e61d811 netfilter-persistent[524]: Warning: skipping IPv4 (no rules to load)
Oct 13 23:43:04 vps-6e61d811 netfilter-persistent[522]: run-parts: executing /usr/share/netfilter-persistent/plugin>
Oct 13 23:43:04 vps-6e61d811 netfilter-persistent[526]: Warning: skipping IPv6 (no rules to load)
Oct 13 23:43:04 vps-6e61d811 systemd[1]: Finished netfilter persistent configuration.
lines 1-15/15 (END)
```

If it is not running, you must enable it by hand:

```bash
sudo systemctl enable netfilter-persistent.service
sudo systemctl start netfilter-persistent.service
```

### Fail2ban

Fail2ban is a tool to prevent intrusions by watching different services logs. We will use it to watch `apache` logs and to protect our `SSH` from brute force attacks.

We must start by enabling it:

```bash
sudo systemctl enable fail2ban --now
```

Now we need to create a local jail and start editing it:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Starts by uncommenting the following lines (to find them quickly type `ctrl` + `w`, then type the string you search, then type `enter`):

- `bantime.increment = true`
- `bantime.multipliers = 1 2 4 8 16 32 64`
- `ignoreip = 127.0.0.1/8 ::1`

Now, we will enable some jails.

A jail looks like this:

```
[sshd]

# To use more aggressive sshd modes set filter parameter "mode" in jail.local:
# normal (default), ddos, extra or aggressive (combines all).
# See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details.
#mode   = normal
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

By default, they are disabled. To enable a jail, just add `enabled = true` after the last parameter:

```
[sshd]

# To use more aggressive sshd modes set filter parameter "mode" in jail.local:
# normal (default), ddos, extra or aggressive (combines all).
# See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details.
#mode   = normal
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
enabled = true
```

There is the list of jails we want to enable:

- `[sshd]`
- `[apache-auth]`
- `[apache-badbots]`
- `[apache-botsearch]`
- `[apache-fakegooglebot]`
- `[apache-modsecurity]`

When you're done, type ++ctrl+s++ to save and ++ctrl+x++ to exit.

Then you need to reload fail2ban for those changes to take effect:

```bash
sudo systemctl restart fail2ban
```

Our server is now fully configured.


## Maintenance

The maintenance is minimal, since `certbot` auto renew all needed certificate automatically and reload `apache2` by its own.

We may want to login to reboot the server every month or so to apply some pending security updates.

The only other real maintenance you would want to do is to deploy a new itrocks version.

To deploy a new update, you must sync the server with the main repository `release` branch. So, first, you must ensure your changes have been pushed to the `release` branch, then all you have to do is a `git pull` on the `VPS`.

To do so, you have two choices :

1) Github action
2) Command line + `SSH`

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

Check that both the target service and the itrocks instance are running:

```bash
sudo systemctl status itrocks.target
sudo systemctl status itrocks@1.service
```

Then you can logout with `exit`.

## Additional notes

This section contains other configuration tips and reminders.

### Iptables (firewall)

If you add a new service that must be reached by the itrocks server or any software that you install on this server, **do not forget to check if a new rule must be added to iptables to open a new port**.

There is the exhaustive list of all open ports (**PLEASE KEEP IT UP TO DATE !**):
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

By default, the panel is publicly available at `http://[server ip]:19999` but our firewall rules will hopefully block it. Since it's a useless config, we will use `apache2` to create a new proxy especially for `netdata`.

Let's start by issuing some self-signed certificates:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```

Now we need to create a new `apache2` configuration:

```bash
sudo nano /etc/apache2/sites-available/vps1-ovh.netdata.itrocks.com.conf
```

Paste this content into the editor:

```apache
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

We now need to enable this new website and reload `apache2`:

```bash
sudo a2ensite vps1-ovh.netdata.itrocks.com.conf
sudo systemctl reload apache2
```

You can now browse [https://vps1-ovh.netdata.itrocks.com](https://vps1-ovh.netdata.itrocks.com) (once your domain have been setup to point to the server IP, obviously).

Accept the security exception regarding the self signed certificate, specify your username (in our case `netdata`) and the password you typed at the previous step and... voilà.

Metrics everywhere!


### Github automation

To allow for deployment directly from the main github repository in tab `Actions`, you must start by creating three secrets into `Settings` -> `Secrets`.

You must adapt their names to try to describe the new server instance with a meaningful name, **they must be different for each server**):

- `OVH_VPS1_HOST` : host ip or domain (for us it is `156.225.137.143`)
- `OVH_VPS1_USER` : main ssh user (for us it is `ubuntu`)
- `OVH_VPS1_PWD` : ssh pwd for the above user

Then you must create a new file in the `master` branch of the main repository in `.github/workflows`, let's call it `ovh_vps1.prod.yml` with the following content:

```yaml
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

**⚠️ VERY IMPORTANT NOTE**: For the GithubAction to be able to run `npm install` with the right version of node,
you **MUST** provide to the system a way to locate the good `node` binary to execute.

To do so, please follow these few steps:

1) Run the command `which node` to get the current used node path. There, it should be `/home/ubuntu/.nvm/versions/node/16.17.1/bin/node`
2) Edit the file `/etc/envrionment` by running `nano /etc/environment`
3) Add **the path** to the node binary by adding **at the beginning** of the `PATH` variable value `/home/ubuntu/.nvm/versions/node/v16.17.1/bin:`
4) Type ++ctrl+s++ to save and ++ctrl+x++ to exit.

---------

Commit to the master branch, and voilà, you'll be able to [use your new GitHub Action as explained in the maintenace section](#github-actions), except you obviously want to select the action you just added.