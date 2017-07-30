# isso-config
Notes and config relating to a test install of the self-hosted commenting solution, Isso.

## Introduction

Having attempted to install Isso on a clean LAMP server (running on a brand new Ubuntu 16 test VM) last week, and running into a crazy amount of difficulty, I've decided to start from scratch, and will document my procedure below. Hopefully this will help to preserve my sanity! ;-)

## Resources

Website: https://posativ.org/isso/  
Install Guide: https://posativ.org/isso/docs/install/#  
GitHub: https://github.com/posativ/isso  

## Preparation

It's Sunday afternoon, 30th July 2017. Realised that my XenServer was missing a security update from 7th July, so installed that and rebooted. Created a brand new test VM, 1 core, 512mb, 20gb SSD, which mirrors the config of a couple of Digital Ocean droplets that I'm using.

Installed Ubuntu 16 with LAMP, standard file system utils, and SSH, but nothing else. Created a user "jonathan", basic password as it's only available on home LAN, automatic security updates. Connected via XenCenter console.

```shell
ifconfig
sudo reboot now
```

Reserved an address in DHCP and DNS, tested via a browser and confirmed the Apache test page is displayed. Connected via PuTTY.

```shell
sudo apt-get update
sudo apt-get upgrade
```

Take a snapshot of the clean build at this point.

## Installation

### Attempt 1

Installation guide at https://posativ.org/isso/docs/install/ gives the following information.

> If you are running Debian/Ubuntu, Gentoo, Archlinux or Fedora, you can use Prebuilt Packages.

Based on that, I'm trying the following install via the package manager.

```shell
clear
sudo apt-get update
sudo apt-get install isso
```

Noted one warning, as follows.

```shell
Setting up isso (0.9.9-1) ...
adduser: Warning: The home directory `/var/lib/isso' does not belong to the user you are currently creating.
```

That folder has been created as follows, currently empty.

```shell
ls -lp /var/lib/ | grep isso
drwxr-xr-x  2 isso  root    4096 Jun  6  2015 isso/
```

Taking note of the info, but continuing with the guide.

```shell
isso --version
isso 0.9.9
```

The page at https://posativ.org/isso/docs/quickstart/ gives the following introduction.

> Assuming you have successfully installed Isso, here's a quickstart quide that covers the most common setup. 

Step one is creating a configuration file. The command to run Isso specifies the location of the configuration file, so it probably doesn't matter exactly where the file is located. I'd typically lean towards placing the config file under /etc, but since the default config file suggest putting the database in /var/lib/isso/ I'm going to put the config there too. (Was initially tempted to put it under /etc/isso.d but not sure that feels very good.)

```shell
dpkg-query -L isso

ls -lp /etc/isso.d
drwxr-xr-x 2 root root 4096 Jun  6  2015 available/
drwxr-xr-x 2 root root 4096 Jun  6  2015 enabled/

cd /etc/isso.d
ls -ad .*
.  ..
```

Based on the guidance at https://posativ.org/isso/docs/configuration/server/ I'm creating the following basic config file for testing.

```shell
cd /var/lib/isso
sudo nano isso.conf
```

```shell
[general]
dbpath = /var/lib/isso/comments.db
host = http://issotest.kyabram.lan/

[server]
listen = http://localhost:8080/
```

Not too sure from the guide, but guessing that file should be owned by the user isso, so updated that below.

```shell
ls -lp
-rw-r--r-- 1 root root 124 Jul 30 20:11 isso.conf
sudo chown isso:root isso.conf
```

Command to run isso (based on the path to my test config) is as follows.

```shell
isso -c /var/lib/isso/isso.conf run
```

However, initially I'm seeing a database warning, "sqlite3.OperationalError: unable to open database file" and perhaps need to create that manually? Trying the following.

```shell
cd /var/lib/isso
sudo touch comments.db
sudo chown isso:root comments.db
ls -lp
-rw-r--r-- 1 isso root   0 Jul 30 20:20 comments.db
-rw-r--r-- 1 isso root 124 Jul 30 20:11 isso.conf
```

Different warning now, "sqlite3.OperationalError: attempt to write a readonly database" so perhaps the ownership of the folder is an issue, and presumably it needs a lock file, or other temporary files?

```shell
cd ..
sudo chown isso:root -R isso

ls -lp | grep isso
drwxr-xr-x  2 isso  root    4096 Jul 30 20:20 isso/

ls -lp isso
-rw-r--r-- 1 isso root   0 Jul 30 20:20 comments.db
-rw-r--r-- 1 isso root 124 Jul 30 20:11 isso.conf
```

Same error, because it's running as me? Going to temporarily change the file permissions to test that thought.

```shell
sudo chmod 777 -R isso

isso -c /var/lib/isso/isso.conf run
2017-07-30 20:34:14,138 INFO: connected to http://issotest.kyabram.lan/
```

And that looks good, but obviously it's bad practice. Going to continue with guide, but noting that these permissions need to be sorted out properly. Also noted that it's running in the foreground, but presumably that is addressed later.

The guide at https://posativ.org/isso/docs/quickstart/ assumes NGINX as a webserver, but this test VM is running Apache. I'm using the provided config as a starting point and attempting the following in Apache.

```shell
cd /etc/apache2/sites-available
sudo nano 000-default.conf
```

Added the following lines to the Apache config.

```shell
<Location "/isso">
    ProxyPass "http://localhost:8080"
    ProxyPassReverse "http://issotest.kyabram.lan:8080"
</Location>
```

Then enabled mod_proxy and restarted Apache.

```shell
sudo a2enmod proxy proxy_ajp proxy_http rewrite deflate headers proxy_balancer proxy_connect proxy_html
sudo service apache2 restart
```

For testing purposes I've opened up another PuTTY window and I'm running Isso in the foreground there. In my other PuTTY session I'll create a simple HTML page to test the comment script.

```shell
cd /var/www/html
sudo nano comment-test-here.html
```

Added the following HTML to the page.

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Comment Test Here</title>
    <meta name="author" content="Jonathan">
    <meta name="description" content="Simple test page for Isso comments">
    <meta name="keywords" content="isso, comments, test">
  </head>
  <body>
  
    <h1>Comment Test Here</h1>
    <p>There's some text here, then hopefully some comments below.</p>
    <hr />
    
    <script data-isso="//issotest.kyabram.lan/isso/" src="//issotest.kyabram.lan/isso/js/embed.min.js"></script>
    <section id="isso-thread"></section>
    
  </body>
</html>
```

It's not working fully, but in the browser console I can see that "embed.min.js" has loaded. However, looking in the console there is the following warning.

```script
embed.min.js:19 Uncaught ReferenceError: exports is not defined
    at embed.min.js:19
    at embed.min.js:19
(anonymous)	@	embed.min.js:19
(anonymous)	@	embed.min.js:19
```

