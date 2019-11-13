---
layout: post
title:  "A complete Pi based VPN Server for under £20"
permalink: /pi-vpn-server.html
date:   2016-07-30 15:59:40 +0100
comments: true
categories: 
---

Updated 9th November 2019 for Raspbian Buster

![Pi Zero Server](../../../assets/pi_zero_server.jpg "Pi Zero Server")

This article describes how, starting from scratch, to build a complete VPN server
based on a Pi Zero for under £20. The method will also work for other Pi models.

There are many tutorials covering this but some are out of date (pre-Buster) and some
assume a fair amount of knowledge about what is going on and may be tricky for the
uninitiated to follow. I have tried to write this assuming minimum experience on the
part of the reader and have concentrated on what to do rather than explaining the 
concepts of VPN, keys, certificates and so on. To get more background understanding the
excellent tutorial by Lauren Orsini at 
[readwrite.com](http://readwrite.com/2014/04/10/raspberry-pi-vpn-tutorial-server-secure-web-browsing/)
is well worth a read and I learned much from it.

# Pre-requisites

* A Raspberry Pi of any flavour, with a network connection (wifi or ethernet).
* For initial setup either a wired ethernet connection or a display and keyboard. 
To use a wired ethernet connection on a Pi Zero you will need a USB Ethernet adaptor.
* A PSU.
* An SD card with up to date with Raspbian Buster Lite installed, you can 
download Raspbian Lite from 
[here](https://www.raspberrypi.org/downloads/raspbian/) 
and load it onto the card as described in the linked installation guide. In fact it should
be ok with the Desktop version of Raspbian installed but why would you do that?
* A PC on the network to connect to the Pi with an ssh client. This is available
by default on most Linux operating systems. For Windows you may need to install
PuTTY, though I see that Windows 10 may now have an ssh client built in.
* You might want some sort of case to put it in when it is complete

# First Steps, Setup the router

In order to access the VPN from the internet there are two requirements. Firstly the 
IP address of the router must be known.  Your internet connection may have a fixed name
or ip address but more usually a service such as duckdns.org is used to get a fixed
name.  How you set this up will depend on which service you go for.  For the examples
here I have assumed that the host name is my_host_name.duckdns.org.

Secondly you must setup port forwarding for UDP and TCP in your router so that port 1194
is forwarded to the pi.  How you do that is dependent on the router.

# Install Raspbian on the Pi

We want the Pi configured as a headless server, programmed and controlled from a PC. To do this 
follow the instructions in the article [Configure your Pi as a Headless Server]({% post_url 2018-03-30-Configure-pi-as-headless-server %}).
You need to give the Pi a fixed IP address as described in that post.

You should now have a Pi running raspbian that you can login to from your PC

# Install packages

The packages openvpn and easy-rsa need to be installed, so connect to the pi and run

`sudo apt install openvpn easy-rsa`

# Build the server certificates and keys

Run the following commands to create the easy-rsa directory and copy the sample files into it

    sudo mkdir /etc/openvpn/easy-rsa
    sudo cp /usr/share/easy-rsa/* /etc/openvpn/easy-rsa
    
Copy the easy-rsa example vars file

`sudo cp /etc/openvpn/easy-rsa/vars.example /etc/openvpn/easy-rsa/vars`

The default certificate expiry time is ten years for the CA certificate but only three for the others.  If you want to change either of those then edit the copied file `/etc/openvpn/easy-rsa/vars`

`sudo nano /etc/openvpn/easy-rsa/vars`

find the lines setting defining those settings (which are in days)

    # In how many days should the root CA key expire?
    #set_var EASYRSA_CA_EXPIRE      3650
    # In how many days should certificates expire?
    #set_var EASYRSA_CERT_EXPIRE    1080

then to change either of them remove the '#' characters to uncomment the appropriate line and set the values as you desire and use Ctrl+S to save it and Ctrl+X to exit.  So for example to set the certifacte expiry period to 10 years change that line to

    set_var EASYRSA_CERT_EXPIRE    3650

**Prepare to build the certificate and keys**

Create an empty directory structure for the keys, /etc/openvpn/easy-rsa/pki, using the easyrsa command.

    cd /etc/openvpn/easy-rsa/
    sudo ./easyrsa init-pki

**Build the certificates and keys**

Leave all settings at default values when prompted for data below, except to say yes when 
asked about signing the certificate. `gen-dh` in particular may take a very long time, particularly on a low 
powered pi, several cups of coffee and an afternoon nap may well be in order.

    sudo ./easyrsa build-ca
    sudo ./easyrsa gen-req server nopass    # accept the default [server] when prompted
    sudo ./easyrsa sign-req server server
    sudo ./easyrsa gen-dh
    sudo openvpn --genkey --secret ta.key

copy ta.key from /etc/openvpn/easy-rsa to /etc/openvpn/  
copy ca.crt dh.pem from /etc/openvpn/easy-rsa/pki to /etc/openvpn/  
copy server.crt from /etc/openvpn/easy-rsa/pki/issued to /etc/openvpn/  
copy server.key from /etc/openvpn/easy-rsa/pki/private to /etc/openvpn/  

    sudo cp ta.key /etc/openvpn/
    sudo cp pki/{ca.crt,dh.pem} /etc/openvpn/
    sudo cp pki/issued/server.crt /etc/openvpn/
    sudo cp pki/private/server.key /etc/openvpn/
    exit

# Configure OpenVPN

Reboot as a check that all is well so far.  Assuming it is then edit `/etc/sysctl.conf` using

`sudo nano /etc/sysctl.conf`

Find the line

`#net.ipv4.ip_forward=1`

remove the # at the start (to un-comment that line), save the file and exit.

To prepare the server configuration file `etc/openvpn/service.conf` we start with the one from
`/usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz` as follows

    sudo -s
    gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > /etc/openvpn/service.conf
    exit
    sudo nano /etc/openvpn/service.conf
    
Find the line

`dh dh2048.pem`

and change it to

`dh dh.pem`

Uncomment, by removing the # or ; if present at the front, from these lines (which are not together in the file)

    topology subnet
    comp-lzo
    push "redirect-gateway def1 bypass-dhcp"
    tls-auth ta.key 0
    user nobody
    group nogroup
    mute 20
    
In addition find the two lines starting `;push dhcp-option DNS ....`, remove the semi-colons and (if you want)
change the ip addresses to the DNS servers of your choice.

Now we can start the server, it is necessary to start two services
    
    sudo systemctl restart openvpn
    sudo systemctl restart openvpn@service

Run `tail -n 50 /var/log/syslog` which will print the last 50 lines of the log and you should
see a page full of ovpn-server messages. Provided there is nothing there that suggests it is not
starting up then run

    ifconfig
    
and you should see an extra network interface, `tun0` with the inet address 10.8.0.1

If it doesn't work but the logs do not contain anything helpful then try a reboot and start openvpn again.
The log /var/log/daemon.log may have useful information also.

To setup openvpn so that it automatically starts on boot, run

    sudo systemctl enable openvpn
    sudo systemctl enable openvpn@service

# Configure iptables

For the vpn to work, iptables must be configured appropriately to forward data to the correct 
destinations.  I prefer to keep iptables configuration in `/etc/rc.local' which is executed on startup.
To do it this way edit the file
  
    sudo nano /etc/rc.local
    
and this just before the `exit 0` at the end.  Note the last lines below, choose either the eth0 or wlan0 line

```
# flush current iptable rules so this can be run at any time to restore rules
# though note that this may clear rules set elsewhere
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -X

# for vpn
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -s 10.8.0.0/24 -j ACCEPT
iptables -A FORWARD -j REJECT
# if using wired ethernet then include this line
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
# if using wifi then comment out above and uncomment this one
#iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o wlan0 -j MASQUERADE

```

To invoke this manually, in a terminal run

    sudo /etc/rc.local
    
# Make the keys and certificates for the clients
 
For each client device that you want to be able to connect to the VPN (laptop, phone etc) you need
to make a set of keys and certificates.  You have to give each of the sets a name.  In this example
the server is called owl and the client tigger. I am using the name tigger_owl so I can remember 
that these are for tigger to connect to owl  

    cd /etc/openvpn/easy-rsa
    sudo ./easyrsa gen-req tigger_owl
    
This will ask for a PEM pass phrase that the user will have to provide when connecting to the VPN.  If `nopass` 
is added to the gen-req command line then it will not be necessary for the user to provide the password.
It is generally not a good idea to use `nopass`, as that would mean that a malicious user getting hold of the client laptop
or mobile device may be able to access your local network.

Next run the command (replacing tigger_owl with your chosen key name obviously)

    sudo ./easyrsa sign-req client tigger_owl
    
The command will ask you to provide the pass phrase that you entered when creating the CA certificate in the 
section "Build the certificates and keys" earlier.  
This makes, in the pki/issued subdirectory, `tigger_owl.crt` that 
the client will need to connect to the VPN. The client will also need the server key and certificate 
`ca.crt` and `ta.key` that the earlier process created and that we copied to /etc/openvpn.

Many client applications can be given a combined ovpn configuration file rather than the individual files.
In order to construct an ovpn file create a file called ovpn_generate.sh in /etc/openvpn/easy-rsa containing
the script that can be found a the end of this blog.  This can be done using

    cd /etc/openvpn/easy-rsa
    sudo nano ovpn_generate.sh

and pasting the script into that (use Ctrl+Shift+V to copy into nano which is a terminal application). Then save it and exit
from nano.  To make it executable run

    sudo chmod +x ovpn_generate.sh
    
Then to create the ovpn file for the example above where the client is called tigger and the server owl, if the domain name
of the vpn were, for example, my_host_name.duckdns.org, run

    cd /etc/openvpn/easy-rsa
    sudo ./ovpn_generate.sh tigger_owl my_host_name.duckdns.org
    
Repeat the gen-req and sign-req commands (and ovpn_generate if required) for each client.

    
# Testing

Having configured an openvpn client on a PC or device, login to the pi from a local PC (preferably not the one about to be used to
connect to the VPN,but if you have not got another device then it will probably be ok) and run

`tail -f /var/log/syslog`

Then connect from the client device.  The log should show the sequence of connection. If the client cannot connect
then there may be some useful messages there.

That's it. All done.

# Performance

I was concerned as to whether a Pi Zero had the horsepower to provide a useful VPN, however my tests gave a throughput 
rate of 10Mbps which is plenty for my purposes. My Pi uses a Wifi link and my Wifi hub is old and a bit flaky so it may
well be that this limit was imposed by the Wifi and not by the Pi.  All the data has to be received by the Pi and then
re-transmitted so 10Mbps throught the server implies 20Mbps across the Wifi.

# Script for ovpn_generate.sh

```
#!/bin/bash

##
## Usage: sudo ./ovpn_generate.sh key_name server
##        e.g. sudo ./ovpn_generate tigger_owl myhostname.duckdns.org 
##

# Given a name used to identify the key and a server host name or ip address
# it builds the file /etc/openvpn/easy-rsa/pki/key_name.ovpn
# Run with sudo after generating keys and certificates
# see http://blog.clanlaw.org.uk/pi-vpn-server.html for more details of usage

if [ "$EUID" -ne 0 ]
  then echo "Must be run with sudo"
  exit
fi

key_name=${1?"The key name is required"}
server=${2?"The server host name or ip address is required"}

cd /etc/openvpn/easy-rsa
cacert="pki/ca.crt"
client_cert="pki/issued/${key_name}.crt"
client_key="pki/private/${key_name}.key"
tls_key="ta.key"
out_file="pki/${key_name}.ovpn"

if [ ! -e "${cacert}" ]
then
    echo "File ${cacert} not found"
    exit
fi
if [ ! -e "${client_cert}" ]
then
    echo "File ${client_cert} not found"
    exit
fi
if [ ! -e "${client_key}" ]
then
    echo "File ${client_key} not found"
    exit
fi
if [ ! -e "${tls_key}" ]
then
    echo "File ${tls_key} not found"
    exit
fi

echo "Building ${out_file} for ${server} from ${tls_key}, ${cacert}, ${client_cert} and ${client_key}"

cat > ${out_file} << EOF
client
dev tun
remote ${server}
nobind
persist-key
persist-tun
verb 1
port 1194
proto udp
cipher AES-256-CBC
comp-lzo
remote-cert-tls server
key-direction 1
<ca>
EOF
cat >> ${out_file} ${cacert}
cat >> ${out_file} << EOF
</ca>
<cert>
EOF
cat >> ${out_file} ${client_cert}
cat >> ${out_file} << EOF
</cert>
<key>
EOF
cat >> ${out_file} ${client_key}
cat >> ${out_file} << EOF
</key>
<tls-auth>
EOF
cat >> ${out_file} ${tls_key}
cat >> ${out_file} << EOF
</tls-auth>
EOF
```



