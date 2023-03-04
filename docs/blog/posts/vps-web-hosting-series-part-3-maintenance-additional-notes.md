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

So... we did install everything we needed on our VPS to fulfill our client (ITRocks) requirements.
The app is running blazing fast, its secured and all but...

Did you though it was enough? Or did you secretly hoped that there will be more? Maybe both! 

Let's go a step further with this last part. To be fair, this part could have been itself an entire
series, but we've already learned a lot, and what can be done next is... an almost infinite amount of things depending on your use cases.

<!-- more -->

!!! warning

    This is the **third** part of a series:

    1. [VPS & Web Hosting series part 1: Installation](/blog/2023/02/27/vps-web-hosting-series-part-1-installation/)
    2. [VPS & Web Hosting series part 2: Configuration](/blog/2023/02/28/vps-web-hosting-series-part-2-configuration/)
    3. [VPS & Web Hosting series part 3: Maintenance & additional notes](/blog/2023/03/01/vps-web-hosting-series-part-3-maintenance-additional-notes/){ .current-page }

    If you didn't read the [VPS & Web Hosting series part 1: Installation](/blog/2023/02/27/vps-web-hosting-series-part-1-installation/)
    or [VPS & Web Hosting series part 2: Configuration](/blog/2023/02/28/vps-web-hosting-series-part-2-configuration/), please considere
    doing it before reading what follows :wink:

## Additional implicit requirements

Very often when you deal with a client (even if he work in IT), you have to fulfill some requirements that said client
didn't even phrased. Being able to anticipate his/her need is a soft skill that come with experience.

Some of those implicit requirements depends on the client and its project, but at least one of them is
100% applicable everytime you do such a work for anyone:

### Documentation

Documentation is **^^NOT^^** optional. 

<div class="gif">
   <div style="width:100%;height:0;padding-bottom:50%;position:relative;"><iframe src="https://giphy.com/embed/xT0xenYQWe3ev6gyTC" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div>
</div>

What if I told you that I love writing it? And what if a told you that ^^you^^ can learn to love it too? This will be the subject of
another article :wink: For now, just remember that documentation in general is mandatory. Our job is complex, we make decisions all the day,
choose all the day long that solution instead of that other one. Our Tech stacks are growing faster than the whole universe expansion every year.

We just can't afford the cost of undocumented whatever in this battlefield.

#### Why is it important?

No matter who you are and why you ended up installing and configuring this **VPS**, you must create a document
that explain what you did and how. The why is not that important in this kind of documentation, unless explaining
something very tricky and specific to the project itself.

So, creating this documentation is mandatory because:

1. Transmission. Someone else than you will probably have to maintain the server for whatever reason
2. Chaos reduction. They are literally thousands of tools, commands, packages and configuration tweaks that can lead to the same result,
   so knowing what have been done is not just a matter of experience and skills.
3. Highlighting gotchas. Even if you're the only maintainer for the whole project life, if you have to come back three years later, you'll probably
   break something because you'll not remember that because `A`, `B` and the conjunction of `C & D | E`, you must run this command after this one
   if it has not been written somewhere. 
4. Legacy. If you or someone else have to change something, you want this someone to document his/her changes. Without an existing documentation, this change will be forgotten too.
5. Auditability. It attests the work you've done so far, and if someone else breaks something, you'll be glad to demonstrate that it was not you.
6. Reviewing. If **you** misconfigured something, someone will be able to spot it reading your exhaustive documentation. This is probably the best benefit of this list!
7. Proofreading. If you misconfigured something, proofreading your documentation a week or two later will help you spot your error.
8. Aren't there still enough good reasons to write this documentation yet? Do you really need another one? :smile:

All of this come to the last and most evident observation: writing this documentation will **actually** save time & money, both for you and your client/company, in a way or another!

#### How to write it?

**KISS** ([not this one](https://www.youtube.com/watch?v=ZhIsAZO5gl0&ab_channel=KissVEVO){ target="_blank" } but rather **Keep It Simple Stupid**)

Just write in chronological order all commands you ran, every tool you installed, every configuration file you edited. And if you forgot to do it for a while, 
you still can even print your bash history to get a good starting point:

```bash
cat ~/.bash_history
```
{ no-linenums }

!!! tip "A very bad documentation is always better than no documentation at all :wink:"

As an example, this kind of section in your documentation can save you from a painful debugging session:

??? example "Example: documentation section about server's open ports"

    If you add a new service that must be reached by the itrocks server or any software that you install on this server, **do not forget to check if a new rule must be added to iptables to open a new port**.

    There is the exhaustive list of all open ports (**KEEP THIS LIST UP TO DATE !**):
    
    | port    | service  |
    |---------|----------|
    | `22`    | SSH      |
    | `53`    | DNS      |
    | `80`    | HTTP     |
    | `443`   | HTTPS    |
    | `465`   | SSMTP    |
    | `27017` | MongoDB  |

### Monitoring

Being able to monitor a server is one of the most important things. Is this node running smoothly or 
do we reach it's maximum capacity? Should we consider upgrading our VPS? Spawning a new instance?
How many requests by second do we handle? Does our server handle high traffic peaks without downtime?

Monitoring can help us understand what's going on, and, more important, can reassure us in the health of
our online service.

That's why we'll [install Netdata below](#netdata).

### But... what about the *implicit* requirements?

Now, let's see what can be an implicit requirement by giving an additional precision about our study case: 

While speaking with our client, we learnt that he's not comfortable with commandline interface nor SSH. He did want a VPS 
for the freedom of configuration it offers at a cheap price, but will outsource its management to someone else. 

All he wants to be able to do is very basic tasks from SSH once in a while (upgrading ITRocks, rebooting/upgrading the server), so he asked us
to give him a simple script to do it in a blink.

He said us that he likes all things to be available in the same place and one this script at the root of is project's repository. 
And at this occasion, he added that he work with a VSCode extension to manage his GitHub repositories too.

This is exactly what I call an *implicit* requirement.

So... let's see what we can do with that information under our belt :smile:

## Additional configuration

First of all, we already redacted a beautiful documentation as we configured the server. So we'll just have to add the few steps below.

It allows us to go for the next task... monitoring!

### Netdata

Netdata is a powerful monitoring tool that even comes with a panel. Not only is it open source, but it do provide a clean cloud environment
**and** my favourite feature about it is... you can write your own data collector. It means that if our client wants to add metrics tailored to its
needs, he can do it without that many efforts later, and it will integrate itself nicely in his Netdata panel!

The icing on the cake: it is fairly simple to install, just run:

```bash
wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh && sh /tmp/netdata-kickstart.sh
```
{ no-linenums }

By default, the panel is publicly available at `http://[server ip]:19999` but our firewall rules will hopefully block it. 
Since it's not that useful like this, we'll use `apache2` to create a new proxy especially for `netdata`.

Let's start by issuing some self-signed certificates:

```bash
sudo openssl req -x509 -nodes -days 365 \
             -newkey rsa:4096 \
             -keyout /etc/ssl/private/apache-selfsigned.key \
             -out /etc/ssl/certs/apache-selfsigned.crt
```
{ no-linenums }

Now we need to create a new `apache2` configuration:

```bash
sudo nano /etc/apache2/sites-available/vps.netdata.itrocks.com.conf
```
{ no-linenums }

Paste this content into the editor:

```apache title="/etc/apache2/sites-available/vps.netdata.itrocks.com.conf"
<VirtualHost *:443>
    ServerName vps.netdata.itrocks.com

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
sudo a2ensite vps.netdata.itrocks.com.conf
sudo systemctl reload apache2
```
{ no-linenums }

You can now browse [https://vps.netdata.itrocks.com](https://vps.netdata.itrocks.com) (once your domain have been set up to point to the server IP, obviously).

Accept the security exception regarding the self-signed certificate, specify your username (in our case `netdata`) and the password you typed at the previous step and... voilà.

Metrics everywhere!

!!! note "I leave it to you to use `certbot` to get a valid certificate instead :wink:"

### GitHub Actions

GitHub Actions are an amazing tool you should put your hands on ASAP if you didn't yet!

Since our client is used to **VSCode** and not comfortable with **CLI**, lets set up a **GitHub Action** that will be integrated directly in its favourite code editor in
no time with a [plugin like `GitHub Actions`](https://marketplace.visualstudio.com/items?itemName=cschleiden.vscode-github-actions){ target="_blank" }

To allow him to deploy directly from the main GitHub repository in tab `Actions`, you must start by creating three secrets into `Settings` -> `Secrets`.

You must adapt their names to try to describe the new server instance with a meaningful name, **they must be different for each server**):

- `PROD_VPS1_HOST` : host ip or domain (for us it is `156.225.137.143`)
- `PROD_VPS1_USER` : main ssh user (for us it is `ubuntu`)
- `PROD_VPS1_PWD` : ssh pwd for the above user

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
          host: ${{ secrets.PROD_VPS1_HOST }}
          username: ${{ secrets.PROD_VPS1_USER }}
          password: ${{ secrets.PROD_VPS1_PWD }}
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

I leave it to you to create the same kind of **GitHub Action** to upgrade and reboot the server at will :wink:  

## Maintenance

First of all, we could have installed a tool like [Plesk](https://www.plesk.com/){ target="_blank } or [CPanel](https://cpanel.net/){ target="_blank }. 
Note that it's proprietary/licensed with a paid version.

For our job, I feel like a plesk panel is too much for our case: our client just need to push a new release in production, to upgrade the server
or just to reboot it.

The maintenance of our **VPS** is minimal, since `certbot` auto-renew all needed certificate automatically and reload `apache2` by its own.
We may want to log in to reboot the server every month or so to apply some pending security updates.

The only other real maintenance you would want to do is to deploy a new `ITRocks` version.

To deploy a new update, you must sync the server with the main repository `release` branch. 
So, first, you must ensure your changes have been pushed to the `release` branch, then all you have 
to do is a `git pull` on the **VPS**.

To do so, you have two choices :

1. The Wonderful GitHub Action we just set up at previous step
2. Command line + **SSH**

### GitHub Actions

The easy way.

Go to the main repository. In the `Actions` tab, selects `Deploy to VPS1 (production)`, then click the `run workflow` button, then the second `run workflow` button.

Wait for a minute or two for the process to end and... voilà. The server has been synchronized with the `release` branch!

And if you're a **VSCode** user, just install the [`GitHub Actions` plugin](https://marketplace.visualstudio.com/items?itemName=cschleiden.vscode-github-actions){ target="_blank" } to let magic
happen!

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

Check that both the target service and the **ITRocks** instance are running:

```bash
sudo systemctl status itrocks.target
sudo systemctl status itrocks@1.service
```
{ no-linenums }

Then you can log out with `exit`.

## Conclusion

This is the end of this **VPS & Web Hosting tutorial series**. I hope that it helped you to have a wider view of what can be done and
how to approach this kind of configuration process.

If you have any question, any observation or any correction, feels free to comment :smile: