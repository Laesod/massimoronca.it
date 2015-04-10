---
title: Start running your own DNS server
date: "2014-02-10"
description: Or why you should be running your own instance of a DNS server, it's easier than you think.
source_name: MIKAMAYHEM
source_url: http://dev.mikamai.com/post/76415968842/start-running-your-own-dns
tags:
- running your private DNS
- DNS server
- DnsMasq
- pow and powder alternatives
---

A common problem in web development is simulating the final environment and, specifically, running your apps in their own private domain.  
One solution is editing the `/etc/hosts` every time you need to add a new domain, but this can quickly become a very tedious process.  
If you work on OS X, you probably have heard of [Pow](http://pow.cx/), but if you only need the domain resolution and don't use Rails, it is probaly overkill to install a full featured application server just to create some dev domain.
[MAMP Pro](http://www.mamp.info/en/mamp-pro/index.html?utm_medium=twitter&utm_source=twitterfeed) offers domain resolution too, but it's not free.  

There is, however, another solution: running your own instance of a DNS server.  
What you need it's a copy of [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) and *stop worrying and love the shell.*.

First of all install dnsmasq and put it in autostart

```bash
brew install dnsmasq  
sudo cp -v $(brew --prefix dnsmasq)/*.plist /Library/LaunchDaemons
```

Configure the dns intance

```bash
cat <<- EOF > $(brew --prefix)/etc/dnsmasq.conf
# IPV4
address=/dev/127.0.0.1
# IPV6, otherwise virtual hosts in Maverick won't work
address=/dev/::1
listen-address=127.0.0.1
EOF
```

Then we configure the resolvers for all domains and create the one for the `.dev` suffixes


```bash
sudo mkdir -p /etc/resolver
# .dev domains
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/dev'
# universal catcher
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/catchall'
sudo bash -c 'echo "domain ." >> /etc/resolver/catchall'
```


Set localhost as main DNS server, unfortunately you can't automatically prepend localhost to the list of DNSs your DHCP assigned to you.  
You have to change it manually and put 127.0.0.1 on top of the list.

```bash
networksetup -setdnsservers Ethernet 127.0.0.1
networksetup -setdnsservers Wi-Fi 127.0.0.1
```

If everything's ok, running `scutil --dns` should return something like this

```bash
resolver #8
domain   : dev
nameserver[0] : 127.0.0.1
flags    : Request A records, Request AAAA records
reach    : Reachable,Local Address
```

Now you can start dnsmasq

```bash
# start dnsmasq
sudo launchctl -w load /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist

# make some tests
$ ping -c 1 anyhostname.dev
PING anyhostname.dev (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.058 ms

$ dig anyhostname.dev
;; ANSWER SECTION:
anyhostname.dev.	0	IN	A	127.0.0.1
;; Query time: 3 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)


$ dig google-public-dns-a.google.com @127.0.0.1
;; ANSWER SECTION:
google-public-dns-a.google.com.	25451 IN A	8.8.8.8
;; Query time: 30 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)

# cache will speedup subsequent DNS queries
$ dig google-public-dns-a.google.com @127.0.0.1
;; Query time: 0 msec
```

Bonus: if you run Apache on your dev machine, you can easily setup the *one virtual host to rule them all* through [mass vistrual hosting](http://httpd.apache.org/docs/2.4/vhosts/mass.html).  

Edit `/etc/apache2/extra/httpd-vhosts.conf` and add this configuration, every `.dev` domain will point to a folder with the same name, but without the extension, in `~/Sites` folder.  
For example, `myapp.dev` will point to `~/Sites/myapp`.

```apache
<VirtualHost *:80>
    ServerAdmin you@localhost
    ServerName anyhostname.dev
    ServerAlias *.dev

    UseCanonicalName Off
    # %1.0 is the domain without the extension, in this case
    # everything before .dev
    # more info: http://httpd.apache.org/docs/2.4/mod/mod_vhost_alias.html
    VirtualDocumentRoot /Users/[your_login]/Sites/%1.0
    ErrorLog "/var/log/apache2/dev_hosts_error_log"
    CustomLog "/var/log/apache2/dev_hosts_access_log" common

    <Directory /Users/[your_login]/Sites/*>
        AllowOverride All
    </Directory>
</VirtualHost>
```

You can finally reload Apache configuration with `sudo apachectl graceful`.
