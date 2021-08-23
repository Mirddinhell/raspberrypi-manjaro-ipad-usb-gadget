# raspberrypi-manjaro-ipad-isb-gadget

How to use Raspberry Pi 4 (Arch Linux ARM aarch64) as a USB gadget from iPad Pro

The Raspberry Pi 4 can use the USB-C port as Ethernet, so when combined with an iPad Pro that has a USB-C port, you can power the Raspberry Pi from the iPad Pro with a single USB-C cable through that cable. You can make an SSH connection. This allows you to quickly fill in difficult parts of the iPad Pro alone (compiling Rust code, running Docker, testing something in a Linux environment) by simply inserting the Raspberry Pi 4 as an accessory.

This article describes the procedure when using Arch Linux ARM (aarch64) as the OS of Raspberry Pi 4. If you are using Raspbian, you should be able to use the original article in the last reference.

In this article, Blink Shell is used as the SSH client on the iPad Pro side, but I think that Termius etc. can be used as long as it is a free application .

Preparing an SD card for booting the Raspberry Pi 4
This time I downloaded the disk image from https://github.com/hsxsix/archlinuxarm-aarch64-rpi and burned it to an SD card.

Those who have an existing Linux environment https://olegtown.pw/Public/ArchLinuxArm/RPi4/rootfs/ download rootfs from https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi- You may also want to create an SD card partition by referring to step 4 .

Actual steps (for macOS)
Insert the SD card into your Mac beforehand.

cd Downloads
curl -LO https://file.hsxsix.com/other/archlinuxarm-aarch64-rpi.img
diskutil list # here SD Check which disk number the card is assigned to (This time /dev/disk2 was)
diskutil umountDisk /dev/disk2
sudo dd if=archlinuxarm-aarch64-rpi.img of=/dev/rdisk2 bs=1m
diskutil eject /dev/disk2
First launch of Raspberry Pi 4
At this point, the Raspberry Pi 4 can only connect to the network by wire, so I connected the display and keyboard to the Raspberry Pi for initial setup until the Wi-Fi setup was complete.

By default, the root user password is root, so log in with that.

First , follow the steps that are written to be the first thing you must do at https://github.com/hsxsix/archlinuxarm-aarch64-rpi .

init_resize
This expands the space of the root file system. After that, it will be restarted automatically, so you can continue working after the restart.

Wi-Fi settings
First, set up Wi-Fi so that you can connect to the Internet.

# Reset the wpa_supplicant related settings provided by this disk image.
rm /boot/wpa_supplicant.conf
rm /etc/wpa_supplicant/wpa_supplicant.conf
systemctl stop wpa_supplicant
systemctl disable wpa_supplicant

# https://wiki.archlinux.org/index.php/WPA_supplicant#Connecting_with_wpa_cli Set with wpa_cli according to
vi /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
# Write the following
# ctrl_interface=/run/wpa_supplicant
# update_config=1
wpa_supplicant -D -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
wpa_cli
wpa_cli You can set it interactively as follows by executing.

> scan
OK
> add_network
0
> set_network 0 ssid "SSID"
> set_network 0 psk "Password"
> enable_network 0
> save_config
OK
> quit
When you are done, create a configuration file on the systemd-networkd side.

vi /etc/systemd/network/25-wireless.network
Write the contents of the file as follows. (See https://wiki.archlinux.org/index.php/Systemd-networkd#Wireless_adapter )

[Match]
Name=wlan0

[Network]
DHCP=ipv4
Activate when you're done.

systemctl start wpa_supplicant@wlan0
systemctl enable wpa_supplicant@wlan0
systemctl restart systemd-networkd
ip a # Make sure you have an IP address on wlan0
Initial setting
Since some settings specific to the disk image used this time have been made, we will return them to the normal state.

vi /etc/pacman.d/mirrorlist
# Geo-IP based mirror selection and load balancing Delete the rows above it so that the mirror server of
vi /etc/pacman.conf
# Delete aur near the end because you don't have to go to see it
The following are general initial settings.

pacman -Syyu # Package database synchronization and package upgrade
passwd # root Change user password
pacman -S sudo
hostnamectl set-hostname Raspberry-beetle # Hostname setting (mDNS uses this hostname + .local, so a shorter one is recommended)
timedatectl set-timezone Asia/Tokyo
visudo
# wheel Enable the following lines to give sudo privileges to users in the group
# %wheel ALL=(ALL) ALL
Created by general user
I created a user named kebo, but read it as your favorite username and run it.

useradd kebo # User created
passwd kebo # Password setting
usermod -aG wheel kebo # wheel Belong to a group (for sudo execution)
mkdir /home/kebo # Home directory creation
chown kebo:kebo /home/kebo # Home directory ownership change
mDNS settings
Use Avahi to enable mDNS. Doing this makes the procedure much easier as the host name + .local can be resolved when accessing the Raspberry Pi 4 from the iPad Pro via a USB-C cable.

pacman -S avahi
systemctl start avahi-daemon
systemctl enable avahi-daemon
To check , open the iPad Pro's Blink Shell

blink> ping Raspberry-beetle.local
PING raspberry-beetle.local (192.168.86.46): 56 data bytes
64 bytes from 192.168.86.46: icmp_seq=0 ttl=64 time=207.943 ms
64 bytes from 192.168.86.46: icmp_seq=1 ttl=64 time=11.601 ms

--- raspberry-beetle.local ping statistics ---
3 packets transmitted, 2 packets received, 33.3% packet loss
round-trip min/avg/max/stddev = 11.601/109.772/207.943/98.171 ms
If a ping response is returned from "hostname + .local" like this, it is successful.

Grow a USB network interface
This is the best part of this article. This will allow the iPad Pro and Raspberry Pi 4 to communicate over Ethernet through a USB-C cable.

echo "dtoverlay=dwc2" | sudo tee -a /boot/config.txt
sudo vi /root/usb0.sh
# https://gist.github.com/kkk669/0c7b044ad9d992a41ed02467bb0d2d71 Please copy the contents of
sudo chmod +x /root/usb0.sh
sudo vi /etc/systemd/system/usb0.service
# https://gist.github.com/kkk669/2ea06a0ca07b16d50e9f5e81e775f045 Please copy the contents of
sudo systemctl daemon-reload
sudo systemctl enable usb0
sudo reboot
After restarting, usb0you can see that the number of network interfaces is increasing as shown below.

$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether dc:a6:32:70:5d:1c brd ff:ff:ff:ff:ff:ff
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether dc:a6:32:70:5d:1e brd ff:ff:ff:ff:ff:ff
    inet 192.168.86.46/24 brd 192.168.86.255 scope global dynamic noprefixroute wlan0
       valid_lft 85755sec preferred_lft 74955sec
    inet6 fe80::758:6bed:b2a3:76f1/64 scope link 
       valid_lft forever preferred_lft forever
4: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:dd:dc:eb:6d:a1 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::35b2:6f4e:23a2:c0ff/64 scope link 
       valid_lft forever preferred_lft forever
SSH connection test
Connect your iPad Pro to your Raspberry Pi 4 with a USB-C to USB-C cable.

Turn off Wi-Fi and Cellular communication on the iPad Pro from Blink Shell

ssh kebo@Raspberry-beetle.local
If you can SSH with something like that, you are successful.

(Bonus) Swap file settings
If you build various things with Raspberry Pi 4, it will be painful if there is no swap, so set the swap file. Here, systemd-swap is used to set it automatically.

sudo pacman -S systemd-swap
sudo vi /etc/systemd/swap.conf
# swapfc_enabled=1 に変更
# swapfc_force_preallocated=1 に変更
# zswap_enabled=0 Changed to (because an error is displayed if it is not supported by the kernel used this time)
sudo systemctl start systemd-swap
sudo systemctl enable systemd-swap
After a while, you can confirm that the swap file has been created with the following command.


swapon --show
