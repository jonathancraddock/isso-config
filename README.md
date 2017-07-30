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

