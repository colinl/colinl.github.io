---
layout: post
title:  "Configure your Pi (Zero upwards) as a headless server"
date:   2016-07-28 17:04:09 +0100
comments: true
categories: 
---

Many Raspberry Pi systems are required to run without a display, keyboard or mouse,
with programming and control of the Pi being achieved via a network connection.
This article describes how, starting from scratch, to setup your Pi (Zero upwards) to operate like this
(i.e. as a Headless Server).

# Pre-requisites

* A Raspberry Pi of any flavour, with a network connection (wifi or ethernet).
* If you are using a Pi Zero you will need a USB Wifi or Ethernet adaptor.
* A PSU.
* An SD card with NOOBS installed. You can either buy a card with NOOBS pre-installed or
download NOOBS and load it onto the card as described [here](https://www.raspberrypi.org/documentation/installation/noobs.md).
* A PC on the network to connect to the Pi.
* For initial setup you will also need to connect a display, keyboard and mouse.
Depending on the model of Pi you may need a USB hub to connect the keyboard and mouse.

# First Steps, Install Raspbian

This is where you will need the display, keyboard and mouse. Connect them and the
wifi adaptor or ethernet connection to the Pi, insert the NOOBS SD card, connect
the power supply and power up.

You should immediately see a multicolour screen for a few seconds.  After a while
the NOOBS install screen should appear.  If you are given a choice of operating systems
to install then select Raspbian, then tell it to Install.  This may take a significant time.

Eventually it should boot into Raspbian.  Dependent on which version of Pi you have the boot
process may take some time and may show a black screen for a while.  Don't panic, just wait.

# Configure Raspbian

Once the Raspbian screen appears you can configure the system.  You may find
that the mouse operation is painfully slow, particularly on a Zero for example, but don't worry
you won't need it for much longer.

If you want to use Wifi then click on the Network icon in the top right hand corner and select
the network you want to connect to, you will be prompted for the key. If anyone knows how to
bring up the connection app using just the keyboard please let me know.

Next select Menu > Preferences > Raspberry Pi Config.  To avoid using the mouse you can use Ctrl+Esc
to open the Menu then cursor keys and Enter to get to the app.

In the System tab set `Host Name` to the name you want the Pi to be known by.  This should be a short name
with no special punctuation. I have a number of Pi Zeros and have used piz001, piz002 etc.  You can also
use a more meaningful name.

Also in the System tab select `Boot to CLI` and select not to `Auto Login`.

In the Interfaces tab check that `SSH` is enabled (it should be by default).

In the Localisation tab set the `Locale` (for me in the UK this is GB) and the `Timezone` (for me this
is UK)

Reboot.  The Pi should start up much quicker as the GUI is not being invoked.

# Check connection from client PC

The Pi should be in terminal mode but not logged in. Enter pi as the user name and
raspberry as the password to login.

Type

`ifconfig`

and Enter.
It will show information on the network connections, and under the section for eth0 (if using a wired
connection) or wlan0 (if using wifi) there should be a line something like

`inet addr:192.168.1.105  Bcast:192.168.1.255  Mask:255.255.255.0`

The first address here (192.168.1.105 in this case) is the current ip address of the Pi.

Connect to the Pi from the PC via SSH. The technique to do this will vary with which
operating system you use on the PC. I believe that those unfortunate enough to have to use
Windows can do this using PuTTY, but I have no experience of this so you will have to look
that up yourself.  Once connected the rest should be identical for Windows or other systems

On a Linux PC to connect open a terminal and run

`ssh pi@<ip address>`

where <ip address> is the address from ifconfig above, for for example:

`ssh pi@192.168.1.105`

You should then be prompted for the password, which is raspberry and then should get the
logged on message, ending with the prompt

`pi@<hostname>:~`

# Set a fixed IP address

You will probably want to set the Pi up with a fixed IP address otherwise it may be given
a different address each time it connects to the network.  There are two ways of doing this.

1. **Set a fixed address in the router.**  This is the preferred technique if you can do it, but many still
have old routers that either do not support this or it is flaky.  If your router does support this
you will need to look in the router documentation to find out how.  You will need the MAC address of
the network interface in the Pi, which is shown in the output from ifconfig in the first line for the
relevant interface, so you will see something like

    `wlan0     Link encap:Ethernet  HWaddr 00:0f:61:0c:84:4e`
    
    Where the MAC address in this case is `00:0f:61:0c:84:4e`

2. **Set the fixed address in the Pi network configuration.**  If you cannot do it in the router then
this is the way.  First check in the router config to find which addresses it will allocate dynamically
to connected devices using DHCP.  Often this will be something like 192.168.1.100 upwards.  Then choose an 
address that is outside that range (such as 192.168.1.99 in this case).  If you choose one inside the 
range then you may end up with two devices with the same address which is disastrous.
The setting has to be added to the file `/etc/dhcpcd.conf`. You can do this using your 
favourite command line editor.  If you have not got a favourite then you can use nano which is installed 
by default with Raspbian.  This needs to be done with root permissions, which you can do on the PC whilst logged
on to the Pi using the `sudo` command. Run

    `sudo nano /etc/dhcpcd.conf`

    then scroll down to the end and add to the end something like

        interface wlan0
        static ip_address=192.168.1.99/24
        static routers=192.168.1.1
        static domain_name_servers=192.168.1.1

    Obviously if it were eth0 that you were using then replace wlan0 with eth0.  The address against routers and
    domain_name_servers should be that of the router.

Whether you have used option 1 or 2 above, now reboot using

`sudo reboot`

Give it a short while to power up and then you should be able to connect using the new address.

`ssh pi@192.168.1.99`

Using whatever address you have chosen.

If that does not work you still have the keyboard and display connected so you can go back to the Pi directly 
and attempt to work out what is wrong.  For example use ifconfig to see what address you now have.

Once you are able to ssh into the Pi using fixed address you can power down the Pi using

`sudo shutdown -h now'

Give it a few seconds to shutdown, then you can disconnect the display, mouse and keyboard, power it up,
and connect from the PC again.

# Update the Raspbian software

At some point it is a good idea to update all the software to the latest. This will take some
time (dependent on your internet speed and on which model the Pi is), possibly an hour or more,
so you can do this now or leave it till later if you like.  To update:

`sudo apt-get update && sudo apt-get dist-upgrade`

The first command updates the database describing which version of each package is available, and the
second downloads the packages and updates the software.  There has been a problem with updating the bluetooth 
driver on some boards, if the dist-upgrade command finishes with an error then reboot and run the complete command again
and it will likely be ok.

# Change the username

You may well wish to change the username from pi to something else, for security
reasons if nothing else. If not then skip this section, though I strongly recommend you at least
change the password away from the default, as described below.  It is preferable to change the name of the default user, rather 
than just adding a new one, so that the default user retains the UUID and GID 1000.
If you do wish to change it then first you need to add a temporary user as you cannot change the name of
a user whilst logged on as that user.  I have chosen to call the user temp, but you may use what name you like.
To add the temp user and add him/her to the sudo group

    sudo adduser temp
    sudo adduser temp sudo

You will be prompted for information, most of which can be left empty. You will also be prompted
for a password for that user. Next disconnect from the pi using `logout` or `exit` and then reconnect as the 
temp user

`ssh temp@<ip address>`

Check that there are no processes running as the pi user

`ps -u pi`

should just show a heading line and no processes. Assuming that is ok then change the username to, for example,
fred

`sudo usermod -l fred pi`

and rename the home folder

`sudo usermod -d /home/fred -m fred`

Logout then reconnect as the new user, still using the password raspberry

`ssh fred@<ip address>`

It is necessary to change fred's group name (which will still be pi) to fred using

`sudo groupmod --new-name fred pi`

And finally you will want to change fred's password away from the default using the command

`passwd`

Which will prompt for the existing and new passwords.

It is possible to remove the temp user, but I do not generally bother.  There are times when it is
useful to have a second user and the space used is minimal.

That is it, all done.
