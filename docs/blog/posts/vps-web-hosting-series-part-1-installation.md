---
date: 2023-02-27
categories:
  - Bare metal
  - Hosting
  - Sysadmin
links:
  - "Part 2: Configuration": blog/posts/vps-web-hosting-series-part-2-configuration.md
  - "Part 3: Maintenance & Additional Notes": blog/posts/vps-web-hosting-series-part-3-maintenance-additional-notes.md
---

![How to configure a VPS for web hosting?](/assets/images/blog/VPS-&-Web-Hosting-series/part-1.jpg){ .cover }

# VPS & Web Hosting series part 1: Installation

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

This is the first part of a series:

1. [VPS & Web Hosting series part 1: Installation](/blog/2023/02/27/vps-web-hosting-series-part-1-installation/){ .current-page }
2. [VPS & Web Hosting series part 2: Configuration](/blog/2023/02/28/vps-web-hosting-series-part-2-configuration/)
3. [VPS & Web Hosting series part 3: Maintenance & additional notes](/blog/2023/03/01/vps-web-hosting-series-part-3-maintenance-additional-notes/)

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
- The project sources are hosted in a Github **private** repository, with two branches:
    - `main`
    - `release` (the one we want to use on the VPS)

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

And... we've done for the installation process \o/

Now, it's configuration time, in the [second part](/blog/2023/02/28/vps-web-hosting-series-part-2-configuration/)
 of this series :wink: