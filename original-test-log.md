# isso-config

**This document is a copy of the original activity log of my attempts to install isso. Having now got it working, I'm writing this up into a more coherent procedure. Saved here for info only.**

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

### Using apt-get

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

This issue appears to be addressed in this thread: https://github.com/posativ/isso/issues/318

The suggestion from @imphil solved it for my install, taking this JS link https://posativ.org/isso/api/js/embed.min.js and copying it to the web folder. The HTML was modified as follows.

```html
<!DOCTYPE html> <html>
  <head>
    <meta charset="UTF-8">
    <title>Comment Test Here</title>
    <meta name="author" content="Jonathan">
    <meta name="description" content="Simple test page for Isso
comments">
    <meta name="keywords" content="isso, comments, test">
  </head>
  <body>

    <h1>Comment Test Here</h1>
    <p>There's some text here, then hopefully some comments below.</p>
    <hr />

    <script data-isso="//issotest.kyabram.lan/isso/"
       src="//issotest.kyabram.lan/embed.min.js">
    </script>
    <section id="isso-thread"></section>

  </body>
</html>
```

The site comments now appear to be working, without any errors or warnings. Two actions arising from the above:

* Permissions need to be set correctly on database folder
* Isso needs to run in the background, automatically as a service
* Bug in the 0.9.9 Debian package, fixed by manually copying the embed JS script

---

### Continued 31/7/2017

Footnote: With isso running in the foreground I tried adding a reply to an existing comment from yesterday. The following output is from the PuTTY session.

```shell
isso -c /var/lib/isso/isso.conf run
2017-07-31 10:14:19,526 INFO: connected to http://issotest.kyabram.lan/
2017-07-31 10:15:42,198 INFO: comment created: {"website": "https://duckduckgo.com", "author": "Mulder", "parent": 1, "created": 1501492542.179662, "text": "<p>I like your comment!</p>", "dislikes": 0, "modified": null, "mode": 1, "hash": "*************", "id": 3, "likes": 0}
```

Going to take a look at the permissions on /var/lib/isso and noted that for reasons unknown I didn't use the "isso" group yesterday. Trying the following.

```shell
cat /etc/group

[...]
isso:x:119:

sudo usermod -a -G isso jonathan

cat /etc/group | grep isso
isso:x:119:jonathan

cd /var/lib
sudo chown isso:isso -R isso
sudo chmod 660 -R isso
```

Logging out and back in to pick up group changes. But, with that permission change, back to an error, so trying 774 instead.

```shell
isso -c /var/lib/isso/isso.conf run
2017-07-31 10:32:11,676 ERROR: No website(s) configured, Isso won't work.

cd /var/lib
sudo chmod 774 -R isso

isso -c /var/lib/isso/isso.conf run
2017-07-31 10:33:32,983 INFO: connected to http://issotest.kyabram.lan/
```

Seems ok, so going with that for now.

Created a sub-folder, /var/www/html/blog, and added a new HTML file in there. Confirmed a unique set of comments are pulled through to here. Before any comments are added you see the following message in the browser console. Presume this is due to attempting to pull a non-existant thread into that new page?

```shell
GET http://issotest.kyabram.lan/isso/?uri=%2Fblog%2Fsub-folder.html&nested_limit=5 404 (NOT FOUND)
```

The warning disappears as soon as at least one comment is added.

The configuration guide points to this init file to run Isso as a service: https://github.com/jgraichen/debian-isso/blob/master/debian/isso.init

Looking in /etc/init.d I note that the above script is already in place, with X attribute, and just needs to be automatically run on startup. Looks like the path to the config file is stored in an environment variable, $ISSO_CONF_FILES_DIR, and also I probably should have used a ".cfg" extension as that's been hardcoded in the script.

> I also take back my earlier comment about it not mattering where the config file goes. Looks like my earlier instinct to put it in /etc/isso.d was closer to being correct. The default folder looks like its supposed to be: /etc/isso.d/enabled

I'll move the config ini file to /etc/isso.d/enabled from it's current location at /var/lib/isso.

```shell
sudo mv /var/lib/isso/isso.conf /etc/isso.d/enabled/isso.cfg
sudo nano sudo nano /etc/default/isso

# Use "yes" (without quotes) to make /etc/init.d/isso start the Isso service.
# The Gunicon WSGI server is required.
START_DAEMON=yes
# IP address and TCP port used by Gunicorn
ISSO_ADDRESS_PORT=127.0.0.1:8000
# Path to load isso configuration files from.
ISSO_CONF_FILES_DIR="/etc/isso.d/enabled"

sudo update-rc.d isso defaults
sudo reboot now
```

And... that's not working. But, confirmed that the setup still works ok if manually run with the following.

```shell
isso -c /etc/isso.d/enabled/isso.cfg run
```

Looking at the syslog, I see the following.

```shell
Jul 31 14:50:20 issotest systemd[1]: Listening on isso socket.
Jul 31 14:50:20 issotest systemd[1]: Started isso commenting system.
Jul 31 14:50:20 issotest gunicorn[1623]: 2017-07-31 14:50:20,964 WARNING: unable to dispatch u'/etc/isso.d/enabled/isso.cfg', no 'name' set
Jul 31 14:50:20 issotest gunicorn[1623]: 2017-07-31 14:50:20,966 WARNING: unable to dispatch u'/etc/isso.d/enabled/isso.cfg', no 'name' set
```

Modified the isso.cfg file as follows. (Noted that the guide at https://posativ.org/isso/docs/configuration/server/ says that name is not used unless dispatching multiple sites?)

```shell
; Basic config on "issotest" local test VM
; Created - 30/7/2017
; ^- added general and server config

[general]
dbpath = /var/lib/isso/comments.db
host = http://issotest.kyabram.lan/
name = isso

[server]
listen = http://localhost:8080/
```

Now looking at the tail of syslog I see the following.

```shell
Jul 31 14:59:02 issotest systemd[1]: Listening on isso socket.
Jul 31 14:59:02 issotest systemd[1]: Started isso commenting system.
Jul 31 14:59:02 issotest gunicorn[1718]: 2017-07-31 14:59:02,651 INFO: connected to http://issotest.kyabram.lan/
Jul 31 14:59:02 issotest gunicorn[1718]: 2017-07-31 14:59:02,675 INFO: connected to http://issotest.kyabram.lan/
```

I'm unclear why that final line is repeated, and although it sounds promising, the comments are still not working. Putting that aside for now, I'm going to try this approach:

https://stiobhart.net/2017-02-24-isso-comments/

...using systemd.

```shell
sudo nano /etc/systemd/system/isso.service
```

Trying the following settings.

```shell
[Unit]
Description=Isso Comments
After=network.target

[Service]
User=isso
Group=www-data
WorkingDirectory=/var/lib/isso
LimitNOFILE=8192
ExecStart=isso -c /etc/isso.d/enabled/isso.cfg run
Restart=on-failure
StartLimitInterval=600

[Install]
WantedBy=multi-user.target
```

Then enable the service.

```shell
sudo chmod 755 isso.service
sudo systemctl enable isso.service
sudo systemctl start isso.service
```

And... that's going nowhere either and don't know systemd well enough to have any idea what's wrong with it! So, going to try this approach with "supervisor" that sounds much nicer.

http://blog.pythonity.com/how-to-use-isso.html

```shell
sudo apt-get install supervisor
sudo nano /etc/supervisor/conf.d/isso.conf

[program:isso]
command = isso -c /etc/isso.d/enabled/isso.cfg run
user = isso
autostart = true
autorestart = true
stdout_logfile = /var/lib/isso/supervisor.log
redirect_stderr = true
environment = LANG=en_US.UTF-8,LC_ALL=en_US.UTF-8

sudo supervisorctl reread && sudo supervisorctl update

sudo supervisorctl start isso
```

Confirmed that the comments are now loading correctly on my test pages, so need to reboot and see if it runs automatically?

```shell
sudo reboot now
```

Had to reload the web page a couple of times (warnings in console) but that's probably a cache issue in chrome. Certainly seems to be working ok now.

Going to leave this for now and carry out further testing later. :-)

---

Installed phpLiteAdmin to have a look at the database, but won't open the file. Trying the following.

```shell
apt-get install php7.0-sqlite3
```
