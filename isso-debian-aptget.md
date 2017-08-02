# Introduction

This procedure is based on the activity log saved here: [Original Install Activity Log](original-test-log.md). It's intended as a guide/checklist for my future reference, but currently not tested on a production server, only on a test VM. The permissions and ownerships described below are too loose for a production environment - I just want to see how it works.

## Description of server

After the actions logged in the document linked above, I reverted to a snapshot of a brand new, clean Ubuntu 16.04 build. It's a basic LAMP server with SSH, and nothing else installed. For the purposes of my testing it's on my home network, AKA **kyabram.lan** and the machine name is **issotest**, hence any later references to **issotest.kyabram.lan**, etc.

## Install Isso Using Package Manager

Check for updates.

```shell
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get reboot now
```

No updates pending now, so installing isso.

```shell
$ clear
$ sudo apt-get install isso
$ isso --version
isso 0.9.9
```

Create isso subfolder alongside /var/www/html. I also want to use this folder with phpLiteAdmin.

```shell
$ cd /var/www
$ sudo mkdir isso
$ sudo chown isso:www-data -R isso
$ sudo chmod 774 -R isso
$ sudo chown www-data:www-data -R html
$ sudo chmod 775 -R html
$ sudo usermod -a -G www-data jonathan
$ exit
```

Log back in and install phpLiteAdmin.

> Find current version at [phpLiteAdmin](https://bitbucket.org/phpliteadmin/public/downloads/).

```shell
$ clear
$ cd /var/www/html
$ sudo apt install unzip
$ wget https://bitbucket.org/phpliteadmin/public/downloads/phpLiteAdmin_v1-9-7-1.zip
$ unzip phpliteAdmin_v1-9-5.zip
$ rm phpLiteAdmin_v1-9-7-1.zip
$ mv phpliteadmin.config.sample.php phpliteadmin.config.php
$ nano phpliteadmin.config.php
```

Edit the database path.

```shell
//directory relative to this file to search for databases
$directory = '../isso';
```

Visited http://issotest.kyabram.lan/phpliteadmin.php and confirmed that I can administer databases in that folder.

For the purposes of this test install, I'm putting the configuration file with the database.

```shell
cd /var/www/isso
nano isso.cfg
```

Basic configuration file as follows.

```shell
[general]
dbpath = /var/www/isso/comments.db
host = http://issotest.kyabram.lan/

[server]
listen = http://localhost:8080/
```

Straighten out the ownership.

```shell
$ cd ..
$ sudo chown isso:www-data -R isso
$ sudo chmod 774 -R isso
```

It should now be possible to manually run isso with the following command.

```shell
$ isso -c /var/www/isso/isso.cfg run
2017-08-02 17:06:22,375 INFO: connected to http://issotest.kyabram.lan/
```

Exit with <ctrl>+<c> and complete the Apache config and create a test page.

### Apache

Enable proxy with the following command.

```shell
$ sudo a2enmod proxy proxy_ajp proxy_http rewrite deflate headers proxy_balancer proxy_connect proxy_html
$ cd /etc/apache2/sites-available
$ sudo nano 000-default.conf
```

Append the following lines to the end of the configuration.

```shell
<Location "/isso">
    ProxyPass "http://localhost:8080"
    ProxyPassReverse "http://issotest.kyabram.lan:8080"
</Location>
```

Apache needs a restart.

```shell
$ sudo service apache2 restart
```

### Test Page
