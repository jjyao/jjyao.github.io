---
layout: post
title: "You don't exist, go away!"
date: 2017-08-30 21:10:41 -0700
comments: true
categories: Debug
---

SSH asks me to go way because of ntp clock skew

<!-- more -->

## Scenario
When I try to ssh to a remote server from my linux desktop, I get:

```
[jyao@localhost]$ ssh eng-portal
You don't exist, go away!
```

Then I realize that my shell prompt changes to `[I have no name!@localhost]$`. Apparently, `whoami` stops working:

```
[I have no name!@localhost]$ whoami
whoami: cannot find name for user ID 16195
```

Another thing I find accidentally is that if I disconnect my desktop from internet, `whoami` works again. So my guess is that `whoami` tries to get name from a remote server if there is internet connection. Otherwise, it falls back to use a local database which contains the correct data. My first theory is that the data on the remote server is corrupted. To figure out the remote server that `whoami` talks to, I run `strace`:

```
[I have no name!@localhost]$ strace -f -e trace=network -s 10000 whoami
...
connect(3, {sa_family=AF_FILE, path="/var/lib/likewise/.lsassd"}, 110) = 0
...
```

So `whoami` talks to `lsassd` daemon which then talks to `Active Directory` server. Based on my first theory, it looks like the data on the `Active Directory` server is corrupted. To confirm this, I run `lw-find-user-by-id`:

```
[I have no name!@localhost]$ /opt/likewise/bin/lw-find-user-by-id 16195
Clock skew detected with active directory server
```

Hmm, this means that my first theory is wrong. To confirm the clock skew, I run `ntpq`:

```
[I have no name!@localhost]$ ntpq -p
remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*lmn1-d1-infra01 10.251.184.29    2 u   44   64  177    1.612  -364830 129.863
+lmn1-d1-infra02 10.251.184.29    2 u   47   64  177    1.512  -364831 129.060
```

It turns out that the desktop clock is off by 6 minutes. Now the question arises: why `ntpd` fails to sync up the correct time with the ntp server after the clock skew happens? After googling, I find that `ntpd` indeed tries to fix the clock skew but just in a slow speed (https://serverfault.com/a/608157). It takes `ntpd` more than 1 week to fix 6 minutes skew.

## Solution
Force `ntpd` to do sync up using `-g` option:

```
// disconnect from internet first as sudo needs whoami to be working
[I have no name!@localhost]$ sudo service ntpd stop
// connect to internet
[I have no name!@localhost]$ sudo ntpd -gq
[I have no name!@localhost]$ sudo service ntpd start
```
