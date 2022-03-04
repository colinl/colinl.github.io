---
layout: post
title:  "Run node-red and Mosquitto on Android with Termux"
permalink: /node-red-android.html
date:   2018-05-31 09:00:00 +0100
comments: true
categories: 
---

This describes how to run node-red and the MQTT server on Android, using
Termux.

# Pre-requisites

An Android device with an internet connection.  I used a Moto G2
running stock Android 6.0

# Install Termux

Termux is an app that provides a Linux environment running on Android.  Install
from the Play Store.  When opened it should provide a terminal interface to 
the Linux system.

See [https://termux.com/touch-keyboard.html](https://termux.com/touch-keyboard.html)
for how to enter Ctrl chars etc.
In particular Ctrl-C is Vol Down+C and Up arrow is Vol Up+W

# Allow Termux to access the device filesystem [optional]

I found it useful to allow Termux to access the Android filesystem, so that it
became easy to move files between the device and the PC by plugging it into the 
PC USB.  To do this, in Termux, run
```
termux-setup-storage
```
This setups up a folder `storage` in termux which is a symlink to 
`/storage/emulated/0`. Run
```
ls -al storage
```
to see what is there.  For example `storage/downloads` is the Download folder.

# Install an editor

You will need an editor for editing configuration files and such like. `nano` is
the simplest. To install it using the Termux package manager, which is a
wrapper round apt, run
```
pkg install nano
```
If you prefer another editor then install that instead.

# Install ssh server [optional]

If you are like me then as soon as possible you will want to install the
ssh server in Termux so that you can connect to the Linux system from your PC
so you don't have to laboriously enter terminal commands using the touch screen.
Install the ssh server.
```
pkg install openssh
```
The ssh server only allows connection using keys rather than a username/password
so you need the public key from your client (the PC).  This will likely be in
the `.ssh` folder on the PC and will be called `id_rsa.pub`.  If you haven't got
a public key then you can generate this on the PC using
```
ssh-keygen -t rsa
```
This needs to be copied to the Android device. This can be done by connecting it to
the PC using a USB cable then using the file manager on the PC to move the file
into the Download folder on the device.  Alternatively it could be picked up
directly from the PC using scp on the device.  The key then needs to be added to the 
authorized_keys file on the device which can be done using
```
cat storage/downloads/id_rsa.pub >> .ssh/authorized_keys
```
Now start the ssh server on the device using
```
sshd
```
To connect to the device using ssh it needs to be accessible on the local 
network and you need to know its IP address.  In termux run
```
ifconfig
```
Look for the section starting wlan0 (assuming you have connected via wifi) and 
in that section you should see something like
```
inet 192.168.1.105
```
which is the IP address.  I like to allocate a fixed IP address so that 
the device always has the same address but I am not going to cover that here.

To connect from the PC you also need an ssh client.  On Linux this is likely to
be ssh.  On Windows you will may have to use something like PuTTY.  Here I will
assume that ssh is the command.  So on the PC run
```
ssh -p 8022 <IP>
```
Where `<IP>` is the IP address, so for example
```
ssh -p 8022 192.168.1.105
```
and you should be connected.

#### Start ssh server automatically

The server can be started automatically be creating a file called `.profile` and 
putting in there
```
# start ssh daemon if not already running
#if ! pgrep sshd >/dev/null 2>&1; then
#    sshd
#fi
```
That file will be executed when Termux is opened and will start the server
provided it is not already running.

Most of the following can now be run on the PC connected to the device.

# Install Mosquitto
```
pkg install mosquitto
```
Create a configuration file for mosquitto 
```
cd ~
mkdir .mosquitto
nano .mosquitto/mosquitto.conf
```
and put in there
```
persistence true
autosave_interval 300
persistence_location /data/data/com.termux/files/home/.mosquitto/		(note, trailing / is mandatory)
log_dest file /data/data/com.termux/files/home/.mosquitto/mosquitto.log
```
If you don't want to use retained topics then you don't need the first three lines.

Mosquitto is then started using
```
mosquitto -c /data/data/com.termux/files/home/.mosquitto/mosquitto.conf -d
```
which will start it in the background, with the log going to 
`.mosquitto/mosquitto.log`

Running the command 
```
tail -f .mosquitto/mosquitto.log
```
will show that file in the terminal, and will show new entries as they appear,
which can be useful if all is not working as expected.  To get more verbose 
logging add `-v` to the mosquitto command line.

#### Start mosquitto automatically

As for the ssh server, mosquitto can be started automatically be adding to
the file `.profile` 
```
# start mosquitto in daemon mode provided not already running
if ! pgrep mosquitto >/dev/null 2>&1; then
    mosquitto -c /data/data/com.termux/files/home/.mosquitto/mosquitto.conf -d
fi
```

