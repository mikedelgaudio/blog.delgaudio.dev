---
title: "How to Setup UFW on Raspberry Pi"
date: 2021-04-08T20:34:10-04:00
draft: false
tags: ["raspberry-pi", "linux", "easy"]
categories: "networking"
---

This tutorial demonstrates how to configure a Raspberry Pi with UFW protection. UFW (Uncomplicated Firewall) is a program to manage simple network firewall rules in numerous Linux distros. Many users utilize UFW to block or allow specific transfer protocols or IP addresses in their Linux environment.

## Prerequisites
To follow along in this tutorial, you will need the following:

- basic Linux command line knowledge
- basic networking knowledge
- a configured Raspberry Pi running on your network

## Login and Update Raspberry Pi
Login to your Raspberry Pi with the user you created for the system

Update to the latest stable packages, using the snippet below:

```posix
sudo apt-get update && apt-get upgrade -y
```

The update may take a while; make sure to wait until the process finishes.

## Install and Enable UFW

1. To install UFW run:
```posix
sudo apt-get install ufw
```

2. Check if UFW is installed with `sudo ufw status verbose` and see the following output:
  ```posix
  $ sudo ufw status verbose
  Status: inactive  
  ```

3. Enable the firewall rules using `sudo ufw enable` and the output will change:
  ```posix
  $ sudo ufw enable
  Firewall is active and enabled on system startup
  ```

***Note: Once UFW is enabled, be careful adding deny rules that can block access to your Raspberry Pi.***

## Configure Optional Firewall Rules
UFW applies the following default rule set to the firewall once enabled:

```posix
# Defaults:
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
```

### Modify a default by altering which connections are allowed or denied:

```posix
# Deny outgoing change
$ sudo ufw default deny outgoing

Default outgoing policy changed to 'deny'
```

You may want to tailor these rules for your application requirements.


### Allow an IP address or protocol to access the Pi using `sudo ufw allow from YOUR_IP_OR_PROTOCOL_HERE`

For example, to allow an IP address such as Cloudflare’s DNS 1.1.1.1 `sudo ufw allow from 1.1.1.1` or use the [List of Supported Transfer Protocols][list-of-protocols]

```posix
# Type the exact IP address or URL
$ sudo ufw allow from 1.1.1.1
Rule added

# To allow a transfer protocol, type the desired protocol name as shown below. 
# UFW automatically adds an IPv4 and IPv6 rule.
$ sudo ufw allow https
Rule added
Rule added (v6)
```

### Remove a firewall rule by first determining the index number of the rule with `sudo ufw status numbered`

```posix
# Using our previous example
$ sudo ufw status numbered

Status: active
        To              Action      From
        --              ------      ----
[ 1]    Anywhere        ALLOW       1.1.1.1
....
```

Then remove the rule (for example, the rule at index 1) using `sudo ufw delete 1`, followed by typing `y` and hitting the `Enter` key to accept the changes

```posix
$ sudo ufw delete 1

Deleting:
allow from 1.1.1.1
Proceed with operation (y|n)? y
```

### Block an IP address or protocol from access to the Pi using `sudo ufw deny from YOUR_IP_OR_PROTOCOL_HERE`

For example, to block Cloudflare’s DNS 1.1.1.1: `sudo ufw deny 1.1.1.1`

```posix
 $ sudo ufw deny from 1.1.1.1
 
Rule added
```

In this transfer protocol example, UFW automatically adds an IPv4 and IPv6 block rule

Use the [List of Supported Transfer Protocols][list-of-protocols] for more information


```posix 
$ sudo ufw deny smtp

Rule added
Rule added (v6)
```

### Check all firewall rules `sudo ufw status`
```posix
$ sudo ufw status

Status: active
To          Action      From
--          ------      ----
Anywhere    ALLOW       1.1.1.1
....
```

## Remove UFW
To disable UFW protection, run `sudo ufw disable`
```posix
$ sudo ufw disable
Firewall is stopped and disabled on system startup
```

To delete UFW, disable the firewall as shown above, then run `sudo apt-get remove ufw`

Afterward, to reclaim the storage space, run `sudo apt-get purge ufw`

[list-of-protocols]: https://help.ubuntu.com/community/UFW
