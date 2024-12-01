---
layout: post
title: Raspberry Pi Setup Script
subtitle: Automate tedious pi setup
tags: [raspberrypi, howto]
author: Matt Himrod
excerpt_separator: <!--more-->
---

When I set up a Raspberry Pi, there are a few things that I have to do right out of the gate to make things easier on myself. They involve putting a few files on the sdcard to make the initial setup easier:

* ssh
* wpa_supplicant.conf
* set_up.sh
<!--more-->

The first file is just a blank file that triggers the pi to enable ssh. I set up most of my Raspberry Pis to be headless, so this combined with a reserved IP address in my router allows me to ssh to the Pi without having to first connect it to a monitor and keyboard.

The second file contains the connection information for my WiFi network. You can find more information about this on the Raspberry Pi website (wireless-cli).

The third file is a script that I made to make the initial configuration easier. I find myself doing a lot of things every time I set up a pi, and it was taking me 20-30 minutes to do everything. With this script, I can do it automatically in just a few minutes.

Here's the script. I'll break apart the different sections below:


```#!/bin/bash

apt-get update
apt-get -y upgrade
apt-get -y install git screen
apt -y autoremove

ln -fs /usr/share/zoneinfo/US/Eastern /etc/localtime
dpkg-reconfigure -f noninteractive tzdata

adduser --disabled-password --gecos "Matt Himrod" matthimrod 
adduser matthimrod sudo
echo -e "password\npassword" | passwd matthimrod

echo matthimrod ALL=\(ALL:ALL\) ALL > /etc/sudoers.d/010_matthimrod

cat <<'__SET_UP_MATT__' > ~matthimrod/set_up.sh 
#!/bin/bash

echo ".cfg" >> .gitignore
git clone --bare "git@github.com:matthimrod/dotfiles.git" $HOME/.cfg
mv .bash_logout .bash_logout.orig
mv .bashrc .bashrc.orig
rm .ssh/authorized_keys
/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME config --local status.showUntrackedFiles no
/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME checkout

sudo rm -rf /boot/System\ Volume\ Information/
sudo rm /boot/set_up.sh
rm ~/set_up.sh

__SET_UP_MATT__
chown matthimrod:matthimrod -R ~matthimrod/set_up.sh
chmod 777 ~matthimrod/set_up.sh

/usr/bin/raspi-config
```

The first set of commands updates the software on the Pi, installs git and screen, and removes unneeded packages. This is the biggest timesaver since I don't have to watch the process and run the next command when the previous one finishes.

Next, I set the time zone to US Eastern (or America/New York).

Then I add my user, add it to the sudo group, and set the password.

Now, the next lines drop another script in my home directory to do a few things as my own user. Specifically, I set up the dotfiles Github repository that I discussed in my last post. I also remove this script from /boot and the one from my home directory.

The last thing I do in the "root" script is run raspi-config so that I can change the hostname.
