---
date: 2023-02-28
categories:
  - Bare metal
  - Hosting
  - Sysadmin
links:
  - "Part 1: Installation": blog/posts/vps-web-hosting-series-part-1-installation.md
  - "Part 3: Maintenance & Additional Notes": blog/posts/vps-web-hosting-series-part-3-maintenance-additional-notes.md
---

![How to configure a VPS for web hosting?](/assets/images/blog/VPS-&-Web-Hosting-series/part-2.jpg){ .cover }

# VPS & Web Hosting series part 2: Configuration

In the previous part, we presented the project, installed required packages and even pulled the source
code of our incredible app **ITRocks** into its own folder.

The thing is... the real work haven't been done yet. Be sure that serious stuff will happen in this second part, so stay
with me. This is when it starts to be very interesting! Don't worry though... it's not that hard, I promise :smile:

We must configure our tools and our **VPS** to put our app in production. So go grab a cup of warm coffee,
open a new terminal and follow me into that really exciting phase!

<!-- more -->

!!! warning

    This is the **second** part of a series:

    1. [VPS & Web Hosting series part 1: Installation](/blog/2023/02/27/vps-web-hosting-series-part-1-installation/)
    2. [VPS & Web Hosting series part 2: Configuration](/blog/2023/02/28/vps-web-hosting-series-part-2-configuration/){ .current-page }
    3. [VPS & Web Hosting series part 3: Maintenance & additional notes](/blog/2023/03/01/vps-web-hosting-series-part-3-maintenance-additional-notes/)

    If you didn't read the [VPS & Web Hosting series part 1: Installation](/blog/2023/02/27/vps-web-hosting-series-part-1-installation/), considere
    doing it before reading what follows :wink:

## Configuration

So... let's start to configure `apache2`, `certbot`, `systemctl`, `logrotate`, `iptables` and `itrocks` to get a fully operational production server.

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

??? question "Wait! `o+x`? Isn't it insecure?"

    The `o` permission is for `others` and means litteraly all the operating system users will be able to `execute`
    a folder or a file.

    `nvm` installs a NodeJS version in the current user home directory. It don't install it globaly for all users.
    One of the easiest thing to do to make that node version available for everyone is to give everyone the permission to execute it.

    But why? Why do we want other users to be able to run NodeJS?

    For our `itrocks` user. This user do not need any home directory, and we want him to only have access to the `/home/ubuntu/ITRocks` directory
    and to be able to start the server with the `node` command. Note that it have no read/write permissions in `/home/ubuntu/.nvm`,
    only `execute`.

!!! warning "If you later install a new version of node using nvm, don't forget to add the `o+x` permission to the newly installed NodeJS version!"

Now, we need to add our main user `ubuntu` to the newly created group `itrocks`, because it's `ubuntu` that will perform updates:

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

Test it by checkout to the main branch and checkout back to the release branch, then use `ls` to check all files and folders belongs to the `itrocks` group:

```bash
cd ITRocks
git checkout main
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

```apache title="/etc/apache2/sites-available/app.itrocks.com.conf"
<VirtualHost *:80>
        ServerName app.itrocks.com

        ProxyRequests Off
        ProxyPreserveHost On
        ProxyVia Full
        <Proxy *>
                Require all granted
        </Proxy>

        RewriteEngine On
        # If the client try to connect a WebSocket
        RewriteCond %{HTTP:Upgrade} =websocket [NC]
        RewriteRule /(.*)           ws://127.0.0.1:7001/$1 [P,L]
        # In any other case, reach the HTTP port
        RewriteCond %{HTTP:Upgrade} !=websocket [NC]
        RewriteRule /(.*)           http://127.0.0.1:7000/$1 [P,L]
</VirtualHost>
```

!!! important "For now, SSL related config is ignored, certbot will edit this file for us. We just need to provide him a working config on an insecure VirtualHost."

Now we want to enable our website:

```bash
sudo a2ensite app.itrocks.com.conf
sudo systemctl reload apache2
```
{ no-linenums }

But before running certbot, we need a running server.


### ITRocks

Not only **ITRocks** is an awesome cutting-edge technology with no rival out there, but it follows best practices
too when it comes to security!

As such, there is no sensible configuration information in the git repository. It means that we have to store those configurations
in a safe place... that will be a `conf.env` config file.

### Safe config storage

Let's start by creating a folder and our `conf.env` file in `~/.config`:

```
mkdir ~/.config/itrocks
nano ~/.config/itrocks/conf.env
```
{ no-linenums }

In the editor, we'll past our configuration:

```ini title="~/.config/itrocks/conf.env"
HTTP_PORT=7000
HTTP_INTERFACE=127.0.0.1
WS_PORT=7001
WS_INTERFACE=127.0.0.1
MONGO_DB_HOST=example.mongodb.net
MONGO_DB_PORT=27017
MONGO_DB_NAME=itrocks
MONGO_DB_PATH=DB_PATH="mongodb+srv://itrocks:itrocks@example.mongodb.net/itrocks?retryWrites=true&w=majority"
```

Type ++ctrl+s++ to save and ++ctrl+x++ to exit.

In order to protect it, we need to set up right permissions:

```bash
chmod -R 400 ~/.config/itrocks
```
{ no-linenums }

Now, the file belongs to the `ubuntu` user and is **readonly**, so the `itrocks` user itself can't
read it. This way, we ensure that even if an exploit can be used in the **ITRocks** app to try to read/change the `conf.env` file that stores
sensible data (credentials, api keys, etc.), it won't be able to access it anyway. 

As a reminder, within our configuration, the `itrocks` user will only have access to two things:

- The `/home/ubuntu/ITRocks` folder (recursively) containing sources with **read, write and execute** permissions.
- The `/home/ubuntu/.nvm` folder (recursively) with only the **execute** permission to be able to run node.

!!! warning "About `/home/ubuntu/ITRocks` permissions"

    Since **ITRocks** is fictive, we don't know what it must be able to do with its own source code.
    
    But as a rule of thumb, it should not be able to write. And if it should be able to read/write files,
    you should provide a working directory path in `conf.env` file, and setup all permissions to this
    directory somewhere else in the system.

It can work because it's `systemd` that will read this file and provide declared environment 
variables to the app. It will read it with `root` privileges.

### Systemctl

This is a service manager used to manage the lifecycle of our `itrocks` instance. We will create two files in `/etc/systemd/system/`:

- `itrocks@.service`: a templated unit that will allow us, later, to run multiple instances of **ITRocks** on the same server. When we would need to horizontally scale later.
- `itrocks.target`: the service unit that will decide how much instances to run at a time and manage them for us. Systemd will ensure **ITRocks** is running from server startup to server shutdown and relaunch it if it crashes.

??? question "Why not using `PM2` or `nodemon` instead ?"

    Some people like to use something like [PM2](https://pm2.keymetrics.io/) or [Nodemon](https://nodemon.io/) to *daemonize* a NodeJS application.
    But I'll not stand with them, because I do find them useless. Since we still need
    some init system to make sure they start on boot and restart if they crash. And that's without saying that any init system shipped
    with any linux distro will give you as many (if not more) features than those NodeJS based process managers.
    
    IMHO, they are a useless additional layer that we should get ride off in a vast majority of use-cases :wink: Keeping our (already) huge tech stacks as tiny as possible is probably one of the most underrated effort in our field nowadays.

    That was my 2 cents about it.

Let's start by `/etc/systemd/system/itrocks@.service`.

We will need to know the path to the node binaries:

```bash
which node
```
{ no-linenums }

Should print something like `/home/ubuntu/.nvm/versions/node/v18.14.2/bin/node`. We will use it to define the first systemctl unit in `ExecStart`. So, do not forget to replace this path if it is different in your environment.

Then create the file by typing `sudo nano /etc/systemd/system/itrocks@.service` and pasting the following content into the editor:

```ini title="/etc/systemd/system/itrocks@.service"
[Unit]
Description = "ITRocks instance %i"
Requires = network.target
After = network.target
PartOf = itrocks.target

[Service]
User = itrocks
Restart = always
WorkingDirectory = /home/ubuntu/ITRocks
EnvironmentFile = /home/ubuntu/.config/itrocks/conf.env
ExecStart = /home/ubuntu/.nvm/versions/node/v18.14.2/bin/node index
StandardOutput = append:/var/log/itrocks.%i.instance.log
StandardError = append:/var/log/itrocks.err.%i.instance.log

[Install]
WantedBy = multi-user.target
```

Type ++ctrl+s++ to save and ++ctrl+x++ to exit.

Then we will create `/etc/systemd/system/itrocks.target` by typing `sudo nano /etc/systemd/system/itrocks.target`:

```ini title="/etc/systemd/system/itrocks.target"
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
{ no-linenums }

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
{ no-linenums }

And paste the following content in the editor:

``` title="/etc/logrotate.d/itrocks"
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
{ no-linenums }

Type the maintainer's mail address, select all domains and accepts terms and conditions.

Once certbot executed, we must edit the SSL config to ensure our websockets will be proxied too.

In our case, the apache file was `/etc/apache2/sites-available/app.itrocks.com.conf`, so certbot will generate a file named `/etc/apache2/sites-available/app.itrocks.com.conf-le-ssl.conf`:

```
sudo nano /etc/apache2/sites-available/app.itrocks.com.conf-le-ssl.conf
```
{ no-linenums }

You should see something like this:

```apache title="/etc/apache2/sites-available/app.itrocks.com.conf-le-ssl.conf"
<IfModule mod_ssl.c>
<VirtualHost *:443>
        ServerName app.itrocks.com

        DocumentRoot /var/www/html

        ProxyRequests Off
        ProxyPreserveHost On
        ProxyVia Full
        <Proxy *>
                Require all granted
        </Proxy>

        RewriteEngine On
        # If the client try to connect a WebSocket
        RewriteCond %{HTTP:Upgrade} =websocket [NC]
        RewriteRule /(.*)           ws://127.0.0.1:7001/$1 [P,L] # WS_PORT=7001 is used
        # In any other case, reach the HTTP port
        RewriteCond %{HTTP:Upgrade} !=websocket [NC]
        RewriteRule /(.*)           http://127.0.0.1:7000/$1 [P,L] # HTTP_PORT=7000

        ProxyPassReverse / http://127.0.0.1:7000/

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
{ no-linenums }

And this is the end of it! ITRocks is now reachable on the url https://app.itrocks.com/

### Iptables

Now that we have a running server, let's secure it with ubuntu's `iptables` firewall.

We will create a file in `/home/ubuntu/tools` called `ìptables.config.sh`:

```bash
cd ~/tools
nano iptables.config.sh
``` 
{ no-linenums }

Past this content into the editor:

```sh title="/home/ubuntu/tools/iptables.config.sh"
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

# Allow distant mongodb connection
iptables -t filter -A OUTPUT -p tcp --dport 27017 -j ACCEPT
iptables -t filter -A INPUT  -p tcp --dport 27017 -j ACCEPT

# SMTP Port
iptables -t filter -A OUTPUT -p tcp --dport 465 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 465 -j ACCEPT
```

!!! danger 

    Modifying this file may **lock you out the server**, as well as using some `iptables` command like `iptables flush`.
    Since by default the policy is to reject all, if you flush all configuration, the server will reject everything the 
    **instant** the command is executed, cutting your SSH connection too. 

    This is still temporary at this step, because those changes only happen in memory. If you mess it up,
    restart the server by using your provider's admin panel. 

    If restart doesn't works because the restart command sent by your provider use a blocked 
    port, you'll need to rely on your provider support, or on emergency or recovery procedure he planned.

Then, type ++ctrl+s++ to save and ++ctrl+x++ to exit.

Give execution permission to this newly created file and run it to apply:

```bash
chmod +x iptables.config.sh
sudo ./iptables.config.sh
```
{ no-linenums }

!!! danger 

    Before persisting those rules in the next step, it is very important to test that everything is working fine:

    1. Exit and reconnect in `SSH` to ensure it is still possible
    2. Try to reach itrocks and make it send a mail to one of your addresses (ensure that you receive it)
    3. Login to itrocks and check that your websocket is successfully connected.
    4. Try to run `sudo apt-get update` and `sudo apt-get upgrade` to ensure that the firewall rules do not prevent those commands to execute.

Once all is ok, we can persist `iptables` configuration:

```bash
sudo iptables-save > rules.v4
sudo mv rules.v4 /etc/iptables
```
{ no-linenums }

Check that  `iptables-persistent` is running:

```bash
sudo systemctl status netfilter-persistent.service
```
{ no-linenums }

Should print something like:

```
● netfilter-persistent.service - netfilter persistent configuration
     Loaded: loaded (/lib/systemd/system/netfilter-persistent.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/netfilter-persistent.service.d
             └─iptables.conf
     Active: active (exited) since Thu 2023-02-21 23:43:04 UTC; 19h ago
       Docs: man:netfilter-persistent(8)
   Main PID: 518 (code=exited, status=0/SUCCESS)
        CPU: 7ms

Feb 21 23:43:04 vps-ac78ef7 systemd[1]: Starting netfilter persistent configuration...
Feb 21 23:43:04 vps-ac78ef7 netfilter-persistent[522]: run-parts: executing /usr/share/netfilter-persistent/plugin>
Feb 21 23:43:04 vps-ac78ef7 netfilter-persistent[524]: Warning: skipping IPv4 (no rules to load)
Feb 21 23:43:04 vps-ac78ef7 netfilter-persistent[522]: run-parts: executing /usr/share/netfilter-persistent/plugin>
Feb 21 23:43:04 vps-ac78ef7 netfilter-persistent[526]: Warning: skipping IPv6 (no rules to load)
Feb 21 23:43:04 vps-ac78ef7 systemd[1]: Finished netfilter persistent configuration.
lines 1-15/15 (END)
```
{ no-linenums }

If it is not running, you must enable it by hand:

```bash
sudo systemctl enable netfilter-persistent.service
sudo systemctl start netfilter-persistent.service
```
{ no-linenums }

### Fail2ban

Fail2ban is a tool to prevent intrusions by watching different services logs. We will use it to watch `apache` logs and to protect our `SSH` from brute force attacks.

We must start by enabling it:

```bash
sudo systemctl enable fail2ban --now
```
{ no-linenums }

Now we need to create a local jail and start editing it:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```
{ no-linenums }

Starts by uncommenting the following lines (to find them quickly type `ctrl` + `w`, then type the string you search, then type `enter`):

- `bantime.increment = true`
- `bantime.multipliers = 1 2 4 8 16 32 64`
- `ignoreip = 127.0.0.1/8 ::1`

Now, we will enable some jails.

A jail looks like this:

```ini title="/etc/fail2ban/jail.local"
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

```ini title="/etc/fail2ban/jail.local"
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
{ no-linenums }

Now that our server is fully configured, we need to have a talk about [maintenance and additional notes, tips and advices](/blog/2023/03/01/vps-web-hosting-series-part-3-maintenance-additional-notes/)
in the third and last part fo this series :smile: