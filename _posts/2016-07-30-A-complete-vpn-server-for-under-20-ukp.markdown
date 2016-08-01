---
layout: post
title:  "A complete Pi based VPN Server for under £20"
date:   2016-07-30 15:59:40 +0100
comments: true
categories: 
---

![Pi Zero Server](../../../assets/pi_zero_server.jpg "Pi Zero Server")

This article describes how, starting from scratch, to build a complete VPN server
based on a Pi Zero for under £20. The method will also work for other Pi models.

There are many tutorials covering this but some are out of date (pre-Jessie) and some
assume a fair amount of knowledge about what is going on and may be tricky for the
uninitiated to follow. I have tried to write this assuming minimum experience on the
part of the reader and have concentrated on what to do rather than explaining the 
concepts of VPN, keys, certificates and so on. To get more background understanding the
excellent tutorial by Lauren Orsini at 
[readwrite.com](http://readwrite.com/2014/04/10/raspberry-pi-vpn-tutorial-server-secure-web-browsing/)
is well worth a read and I learned much from it.

# Pre-requisites

* A Raspberry Pi of any flavour, with a network connection (wifi or ethernet).
* If you are using a Pi Zero you will need a USB Wifi or Ethernet adaptor.
* A PSU.
* An SD card with up to date NOOBS installed (which will have Raspbian based on Debian Jessie).
You can either buy a card with NOOBS pre-installed or
download NOOBS and load it onto the card as described [here](https://www.raspberrypi.org/documentation/installation/noobs.md).
* A PC on the network to connect to the Pi.
* For initial setup you will also need to connect a display, keyboard and mouse.
Depending on the model of Pi you may need a USB hub to connect the keyboard and mouse.
* You might want some sort of case to put it in when it is complete

# First Steps, Install Raspbian

We want the Pi configured as a headless server, programmed and controlled from a PC. To do this 
follow the instructions in the article [Configure your Pi as a Headless Server]({% post_url 2016-07-28-Configure-pi-as-headless-server %}).
You need to give the Pi a fixed IP address as described in that post.

You should now have a Pi running raspbian that you can logon to from your PC

# Install packages

The packages openvpn and easy-rsa need to be installed, so connect to the pi and run

`sudo apt install openvpn easy-rsa`

I am using the recently added command `apt` here rather than `apt-get`. You can continue to use `apt-get`
if you are more comfortable with that.

# Build the server certificates and keys

Run the following commands to create the easy-rsa directory and copy the sample files into it

    sudo mkdir /etc/openvpn/easy-rsa
    sudo cp /usr/share/easy-rsa/* /etc/openvpn/easy-rsa
    
Edit the easy-rsa variables file `/etc/openvpn/easy-rsa/vars`
    
`sudo nano /etc/openvpn/easy-rsa/vars`

and adjust the key settings as follows:

    export KEY_SIZE=2048
    export KEY_COUNTRY="your country"
    export KEY_PROVINCE="your province"
    export KEY_CITY="your city"
    export KEY_ORG="your organisation"
    export KEY_EMAIL="your email"
    export KEY_OU="your organisational unit"
    
It doesn't matter much what you put in the fields, it is just to tie the certificate back to the originator.

**Prepare to build the certificate and keys**

We will use `sudo -s` here (which opens a root shell) rather than using sudo on each line as we need to 
set environment variables (from the file vars) for successive commands.  Having entered a root shell
take care to remember to exit it so you don't accidentally do things you shouldn't!

    sudo -s
    cd /etc/openvpn/easy-rsa/
    mkdir keys
    touch keys/index.txt
    echo 01 > keys/serial
    source ./vars
    ./clean-all

**Build the certs and keys**

Leave all settings at default values when prompted for data below. This may take quite a long time.

    ./build-ca
    ./build-key-server server
    ./build-dh
    cd keys
    openvpn --genkey --secret ta.key

copy server.crt server.key ca.crt dh2048.pem ta.key from /etc/openvpn/easy-rsa/keys to /etc/openvpn/

    cp {server.crt,server.key,ca.crt,dh2048.pem,ta.key} /etc/openvpn/
    exit

# Make the keys and certificates for the clients
 
For each client device that you want to be able to connect to the VPN (laptop, phone etc) you need
to make a set of keys and certificates.  You have to give each of the sets a name.  In this example
the server is called owl and the client tigger. I am using the name tigger_owl so I can remember 
that these are for tigger to connect to owl  

    sudo -s
    cd /etc/openvpn/easy-rsa
    source ./vars
    ./pkitool tigger_owl2
    exit    
    
This makes, in the keys subdirectory, `tigger_owl2.crt` and `tigger_owl2.key` that need to be copied to 
the client along with the server key and certificate `ca.crt` and `ta.key` that the earlier process
created.  Repeat the .pkitool line for each client.

# Configure OpenVPN

Disable openvpn (so it does not restart on boot) for the moment

`sudo systemctl disable openvpn`

Reboot as a check that all is well so far.  Assuming it is then edit `/etc/sysctl.conf` using

`sudo nano /etc/sysctl.conf`

Find the line

`#net.ipv4.ip_forward=1`

remove the # at the start (to un-comment that line), save the file and exit.

To prepare the server configuration file `etc/openvpn/server/conf` we start with the one from
`/usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz` as follows

    sudo gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > /etc/openvpn/server.conf
    sudo nano /etc/openvpn/server.conf
    
Find the line

`dh dh1024.pem`

and change it to

`dh 2048.pem`.

In fact it may already be that.

Uncomment, by removing the # at the front, the lines (which are not together in the file)

    push "redirect-gateway def1 bypass-dhcp"
    tls-auth ta.key 0
    user nobody
    group nogroup
    mute 20

Now we can start the server
    
    sudo systemctl start openvpn

Run `tail -n 50 /var/log/syslog` which will print the last 50 lines of the log and you should
see a page full of ovpn-server messages. Provided there is nothing there that looks obviously 
like an error then run

    ifconfig
    
and you should see an extra network interface, `tun0` with the inet address 10.8.0.1

If it doesn't work but the logs do not contain anything helpful then try a reboot and start openvpn again.
The log /var/log/daemon.log may have useful information also.

To setup openvpn so that it automatically starts on boot, run

    sudo systemctl enable openvpn
    
That completes setting up the Pi.

# Setup the router

The flow of data through the VPN is (more or less) as follows:

1. Client sends data addressed to the external IP of the server's router.
2. The router passes it on to the server (Pi).
3. The server decodes it and sends the request on the intended recipient with a return address
(if it is configured as above) of 10.8.0.0
4. The recipient receives the request and sends the reply via the server's router addressed to 10.8.0.0
5. The server's router passes it on to the server
6. The server sends it back to the client.

To achieve the above two things must be setup in the router

1. Port forwarding of port 1194 to the fixed IP of the Pi. Since we have configured the VPN to use UDP I
had expected to only need to forward UDP on port 1194 but I found I had to forward both UDP and TCP. Whether 
that is a bug in my router or is a general requirement I don't know.
2. In order to achieve point 5 setup a static route on the router so anything addressed to 10.8.0.0
is passed to the IP of the Pi. Look for 'Static Route' configuration in the document.

On mine, for the static route I had to set

    Destination   10.8.0.0
    Mask          255.255.255.0
    Gateway       192.168.49.83
    
# Testing

Having installed openvpn on a server (which is not realy the focus of this article, but there are
some notes on this below) logon to the pi from a local PC (not the one about to be used to connect to the VPN)
and run

`tail -f /var/log/syslog`

Then connect from the client device.  The log should show the sequence of connection. If the client cannot connect
then there may be some useful messages there.

# Performance

I was concerned as to whether a Pi Zero had the horsepower to provide a useful VPN, however my tests gave a throughput 
rate of 10Mbps which is plenty for my purposes. My Pi uses a Wifi link and my Wifi hub is old and a bit flaky so it may
well be that this limit was imposed by the Wifi and not by the Pi.  All the data has to be received by the Pi and then
re-transmitted so 10Mbps throught the server implies 20Mbps across the Wifi. Unfortunately I have not got a USB 
Ethernet adaptor so am not able to test it wired.

# Setting up OpenVPN client on Ubuntu

Setting up the clients is not the focus of this article but in case it is useful to anyone 
here are some notes on setting it up on Ubuntu 16.04

`sudo apt install network-manager-openvpn`

* On the Network icon in the top panel select VPN > Configure VPN
* Click Add
* Type OpenVPN
* Create...
* Connection Name: Some Name
* Gateway `<external ip address or domain name of server's router>`
* Authentication TLS
* User certificate `<client_server>.crt`
* CA certificate `<server_ca>.crt`
* Private key `<client_server>.key`
* No password for key

* Advanced:
* On Advanced > General:
* Use LZO compression
* On Advanced > TLS Authentication
* Additional TLS authentication using `<server_ta>.key`, direction 1
* Save

# Setting up OpenVPN client on Android

Again this is not the focus of this article, but in case anyone finds it useful, this is what I did.

I used the app [OpenVPN for Android](https://play.google.com/store/apps/details?id=de.blinkt.openvpn)

First make client keys and certificate as described earlier, in my case I named them phone_owl.crt, etc.

Use [this script](http://codegists.com/snippet/shell/ovpn-writersh_boeingx_shell) to generate an ovpn file
by running

`script.sh <external ip address of server's router> ca.crt phone_owl.crt phone_owl.key ta.key > phone_owl.ovpn`

Copy that file to (for example) the Downloads folder on the Android device.

On the Android device start the OpenVPN app.

* Click the Folder with the down arrow in it in the top panel
* Find and open the ovpn file
* Give the VPN network an appropriate name
* Click the tick in the top panel

You should now be able to connect to the VPN. Delete the ovpn file from Downloads.



