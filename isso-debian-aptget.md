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

### Web root and Isso folder

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

### phpLiteAdmin

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

### Isso Configuration File

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

### Test Isso (manually)

It should now be possible to manually run isso with the following command.

```shell
$ isso -c /var/www/isso/isso.cfg run
2017-08-02 17:06:22,375 INFO: connected to http://issotest.kyabram.lan/
```

Exit with <kbd>Ctrl</kbd>+<kbd>C</kbd> and complete the Apache config and create a test page.

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

Create a test page in /var/www/html.

```shell
$ cd /var/www/html
$ sudo nano comment-test-here.html
```

Use the following html file, or take it from here: [comment-test-here.html](comment-test-here.html)

```html
<!DOCTYPE html> <html>
  <head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Comment Test Here</title>
    <meta name="author" content="Jonathan">
    <meta name="description" content="Simple test page for Isso
comments">
    <meta name="keywords" content="isso, comments, test">

    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap-theme.min.css" integrity="sha384-rHyoN1iRsVXV4nD0JutlnGaslCJuC7uwjduW9SVrLvRYooPp2bWYgmgJQIXwl/Sp" crossorigin="anonymous">
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.12.4/jquery.min.js">
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/js/bootstrap.min.js" integrity="sha384-Tc5IQib027qvyjSMfHjOMaLkfuWVxZxUPnCJA7l2mCWNIpG9mGCD8wGNIcPD7Txa" crossorigin="anonymous"></script>

  </head>
  <body>

  <div class="container">

    <h1>Comment Test Here</h1>
    <p>There's some text here, then hopefully some comments below.</p>
    <hr />

    <script data-isso="//issotest.kyabram.lan/isso/"
       src="//issotest.kyabram.lan/js/embed.min.js">
    </script>

    <section id="isso-thread"></section>

  </div>

  </body>
</html>
```

### Implement "Uncaught ReferenceError" workaround

Copy the following javascript file into /var/www/html/js.

```shell
$ mkdir /var/www/html/js
$ cd /var/www/html/js
$ wget https://posativ.org/isso/api/js/embed.min.js
```

### Run Isso (manually)

Run Isso manually.

```shell
$ isso -c /var/www/isso/isso.cfg run
2017-08-02 17:33:16,623 INFO: connected to http://issotest.kyabram.lan/
```

### Test in a Browser

In my config, I visited http://issotest.kyabram.lan/comment-test-here.html and added a single comment.

![Demo screenshot of test comment](https://jonathancraddock.com/images/test-comment-1.png)

At the command line (isso currently running in foreground) you see the following output.

```shell
$ isso -c /var/www/isso/isso.cfg run
2017-08-02 17:36:22,422 INFO: connected to http://issotest.kyabram.lan/
2017-08-02 17:37:07,456 INFO: new thread 1: Comment Test Here
2017-08-02 17:37:07,479 INFO: comment created: {"website": null, "author": "Arthur Pewty", "parent": null, "created": 1501691827.457397, "text": "<p>First comment on new test build.</p>", "dislikes": 0, "modified": null, "mode": 1, "hash": "9f0076fd038d", "id": 1, "likes": 0}
```

### Run Isso as a Service


