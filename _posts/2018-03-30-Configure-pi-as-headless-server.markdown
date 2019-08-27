---
layout: post
title:  "Configure your Pi (Zero upwards) as a headless server (Buster)"
permalink: /pi-headless-server.html
date:   2018-03-30 09:40:00 +0100
comments: true
categories: 
---

Many Raspberry Pi systems are required to run without a display, keyboard or mouse,
with programming and control of the Pi being achieved from a PC via a network connection.
This article describes how, starting from scratch, to setup your Pi (Zero upwards) to operate like this
(i.e. as a Headless Server).

# Pre-requisites

* A Raspberry Pi of any flavour.
* For initial setup either a wired ethernet connection or a display and keyboard. 
To use a wired ethernet connection on a Pi Zero you will need a USB Ethernet adaptor.
* A PSU.
* An SD card with up to date with Raspbian Buster Lite installed, you can 
download Raspbian Lite from 
[here](https://www.raspberrypi.org/downloads/raspbian/) 
and load it onto the card as described in the linked installation guide.
* A PC on the network to connect to the Pi with an ssh client. This is available
by default on most Linux operating systems. For Windows you may need to install
PuTTY, though I see that Windows 10 may now have an ssh client built in.

# First Steps, initial configuration using wired ethernet connection 

* If you want to do the initial configuration using a wired ethernet connection
then after burning the SD card create a file called ssh on the boot partition in
order to enable ssh. You should be able to do this using the file manager on your PC
as the boot partition is FAT format and can be accessed from any operating system.
* Connect the pi to your network using an ethernet cable.
* If you will be using wifi and the pi does not have onboard wifi then also plug 
in a wifi adaptor.
* Power up.
* Give it a minute to get going.
* From the ssh client on the pc run `ssh pi@raspberrypi.local` and when it asks
for a password enter `raspberry`. The password will not be echoed.
* If it asks for a password but won't accept it then you have probably typed 
it incorrectly (check you have not got caps lock on).
* If instead of asking for a password it says something like 
`ssh: Could not resolve hostname raspberrypi.local: Name or service not known`
then first try unplugging the ethernet cable for ten seconds then plug it back 
in, wait a bit, and try again.  If still no joy then login to your router and find the section 
on DHCP leases, hopefully there
you will find the pi along with the IP address the router has given it. Then run 
`ssh pi@nn.nn.nn.nn` where nn.nn.nn.nn is the IP address, obviously.
* If instead of asking for a password it says cannot connect then make sure you 
haven't got another pi with the same host name (ie computer name), which is raspberrypi,
on the network.  If you have you will either have to disconnect it or change its
host name.
* Once you have managed to login the run `sudo raspi-config` and carry on as described below.

# First Steps, initial configuration using keyboard and display

* Connect a keyboard and Display and power up.
* You should see on the display the power up log scrolling past. Wait for that to 
complete. Eventually it should stop with a login prompt. Don't panic if it seems
to have stopped without the login prompt, on a low powered pi it may stop for 
tens of seconds at a couple of points during the startup.
* Once the login prompt appears type `pi` and Enter.
* A password prompt should appear, enter `raspberry` and Enter. The password is not 
echoed.  You should now be logged on.
* Run `raspi-config` and select Interfacing Options and enable SSH, then 
continue as described below.

# Host and Network Configuration with raspi-config

* Note that raspi-config is a text application, even if you have a mouse connected
you will not be able to use it. Use the cursor up/down arrows to step between selections,
Enter to make a selection and Tab to move to other fields.
* First you should change the default password, select that option and follow the instructions.
* Next setup the network. Change the computer name (Hostname) to something other than
the default of raspberrypi.
* It is probably a good idea to reboot at this point in order to make sure those changes 
have taken (click `Finish` in raspi-config and you will be given an option to reboot).
* Having rebooted you should now be able to login with your new password, if connecting
via ssh then use `ssh pi@host_name.local` where host_name is the name you gave it above.
* run `sudo rasp-config` again.
* Select Localisation Options and check that the correct locale appropriate for your region
for your region has been automatically selected. There will be a * against the
selected one (for me in the UK this is en_GB.UTF8). Then setup the timezone.
* If you want to use wifi to connect then in Network Options select Wifi and follow the 
instructions.  Take care to enter the SSID and key correctly.
When complete exit raspi-config and run `ifconfig`. You should see the wifi interface
`wlan0: ` and on the line below that it should show `inet nnn.nnn.nnn.nnn` where that is the ip 
address the router has allocated it on the wifi. If the inet line does not appear 
then it has not connected to the wifi.
Assuming that it has connected then if you have been using an ethernet connection 
shutdown (using `sudo halt`), remove the
ethernet connection, and power up again. You should now be able to connect again via
ssh but now it will use the wifi connection.
* If you have been using a keyboard and display then you should now be able to connect
from your PC via wifi using ssh as described above in First Steps using an ethernet 
connection.  Assuming that works you can shutdown (`sudo halt`), disconnect the 
keyboard and display and put them away in the cupboard, you should not need them again.

# Optional - Set a fixed IP address

You may wish to set the Pi up with a fixed IP address otherwise it may be given
a different address each time it connects to the network.  There are two ways of doing this.

1. **Set a fixed address in the router.**  This is the preferred technique if you can do it, but many
older routers either do not support this or it is flaky.  If your router does support this
you will need to look in the router documentation to find out how.  You will need the MAC address of
the network interface in the Pi, which is shown in the output from ifconfig in a line below the
header line for relevant interface, so for wifi look below the line for `wlan0: ` and you will 
see something like

    `ether 00:0f:61:0c:84:4e `
    
    Where the MAC address in this case is `00:0f:61:0c:84:4e`

2. **Set the fixed address in the Pi network configuration.**  If you cannot do it in the router then
this is the way.  First determine the IP address of the pi by running `ifconfig`
That will show information on the network connections, and under the section for eth0 (if using a wired
connection) or wlan0 (if using wifi) there should be a line something like

`inet 192.168.1.105  netmask 255.255.255.0  broadcast 192.168.49.255`

The first address here (192.168.1.105 in this case) is the current ip address of the Pi.
Check in the router config to find which addresses it will allocate dynamically
to connected devices using DHCP.  Often this will be something like 192.168.1.100 upwards.  Then choose an 
address that is outside that range (such as 192.168.1.99 in this case).  If you choose one inside the 
range then you may end up with two devices with the same address which is disastrous.
The setting has to be added to the file `/etc/dhcpcd.conf`. You can do this using your 
favourite command line editor.  If you have not got a favourite then you can use nano which is installed 
by default with Raspbian.  This needs to be done with root permissions, which you can do 
using the `sudo` command. Run

`sudo nano /etc/dhcpcd.conf`

then scroll down to near the end where you will find an example static address 
specification. Remove the # characters (which are comment characters causing
the line to be ignored) so you have something like

    interface wlan0
    static ip_address=192.168.1.99/24
    static routers=192.168.1.1
    static domain_name_servers=192.168.1.1 8.8.8.8

Obviously if it were eth0 that you were using then replace wlan0 with eth0.
The address against routers and
domain_name_servers should generally be that of the router, the fallback of 8.8.8.8
will be used if the router cannot supply an address

Save using Ctrl+O then Enter, and exit with Ctrl+X

Whether you have used option 1 or 2 above, now reboot using

`sudo reboot`

Give it a short while to power up and then you should be able to connect using the new address.

`ssh pi@192.168.1.99`

Using whatever address you have chosen.  You should still be able to connect using host_name.local

# Update the Raspbian software

At some point it is a good idea to update all the software to the latest. This will take some
time (dependent on your internet speed and on which model the Pi is), possibly an hour or more,
so you can do this now or leave it till later if you like.  To update:

`sudo apt update && sudo apt full-upgrade`

The first command updates the database describing which version of each package is available, and the
second downloads the packages and updates the software.  There has been a problem with updating the bluetooth 
driver on some boards, if the yupgrade command finishes with an error then reboot and run the complete command again
and it will likely be ok.

# Change the username

You may well wish to change the username from pi to something else, for security
reasons if nothing else. If not then skip this section.  It is preferable to change 
the name of the default user, rather 
than just adding a new one, so that the default user retains the UUID and GID 1000.
If you do wish to change it then first you need to add a temporary user as you cannot change the name of
a user whilst logged on as that user.  I have chosen to call the user temp, but you may use what name you like.
To add the temp user and add him/her to the sudo group

    sudo adduser temp
    sudo adduser temp sudo

You will be prompted for information, most of which can be left empty. You will also be prompted
for a password for that user. Next disconnect from the pi using `logout` or `exit` and then reconnect as the 
temp user

`ssh temp@host_name`

Check that there are no processes running as the pi user

`ps -u pi`

should just show a heading line and no processes. Assuming that is ok then change 
the username pi to, for example, fred

`sudo usermod -l fred pi`

and rename the home folder

`sudo usermod -d /home/fred -m fred`

Logout then reconnect as the new user

`ssh fred@host_name.local`

Finally it is necessary to change fred's group name (which will still be pi) to fred using

`sudo groupmod --new-name fred pi`

It is possible to remove the temp user, but I do not generally bother.  There are times when it is
useful to have a second user and the space used is minimal.

That is it, all done.
