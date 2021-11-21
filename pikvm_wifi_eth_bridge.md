# Raspberry Pi Wifi-to-eth bridge on PiKVM/Arch Linux

I'm planning to use [PiKVM - Open and cheap DIY IP-KVM on Raspberry Pi](https://pikvm.org/) and unfortunately will not be able to use any Ethernet-based connection to the local network and the internet, but have to rely on Wifi. The remotely controlled servers on the other hand only have Ethernet ports, so I need some kind of Wifi to Ethernet bridge to make my setup work. Since the Raspberry Pi already provides both Wifi and a Ethernet port and the Pi will be mostly idle (besides hosting the PiKVM service), this is a nice opportunity to make myself familiar with a bit of networking stuff.

Note that this is done by creating a new **IP Subnet** with **NAT Routing** since the Wifi interface of the Pi doesn't support `WDS/4addr` which would ease setting up a Wifi-Eth bridge substantially.

If you want to use the **same IP subnet**, please consider reading this tutorial using `parprouted`, a Proxy ARP IP bridging daemon (although it is based around Raspberry Pi OS/Debian):

https://willhaley.com/blog/raspberry-pi-wifi-ethernet-bridge/

Other helpful links are:

https://wiki.archlinux.org/title/Network_bridge

https://raspberrypi.stackexchange.com/questions/88954/workaround-for-a-wifi-bridge-on-a-raspberry-pi-with-proxy-arp/88955#88955

https://wiki.debian.org/BridgeNetworkConnections#Bridging_with_a_wireless_NIC

## Step-by-step tutorial

This tutorial is based around a wifi interface called `wlan0`, an eth interface called `eth0` and the IP subnet `10.1.1.0/24`. If you have differently called network interfaces or prefer another IP subnet, feel free to adjust my instructions as needed.

Make sure you're `root` and the filesystem is set to be writable:

    $ ssh root@[PiKVM-IP]
    or when already on the PiKVM
    $ [sudo] su
    $ rw

Install **dnsmasq**:

    $ pacman -S dnsmasq

Apply **DHCP** settings including new IP subnet in `/etc/dnsmasq.conf`:

    interface=eth0
    bind-interfaces
    server=8.8.8.8
    domain-needed
    bogus-priv
    dhcp-range=10.1.1.100,10.1.1.120,12h

Set **iptables** masquerade rule for `wlan0`:

    $ iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE

Save **iptables** ruletable:

    $ iptables-save > /etc/iptables/iptables.rules

Enabling **IPv4 packet forwarding** by creating new **sysctl** config file in `/etc/sysctl.d/` with a name of your choice, for example `10-ipforward.conf`:

    net.ipv4.ip_forward=1

Configure **static IP address** for `eth0` in `/etc/systemd/network/eth0.network`:

    [Match]
    Name=eth0
    
    [Network]
    Address=10.1.1.1/24

Enable startup of **iptables** and **dnsmasq** on boot:

    systemctl enable iptables.service
    systemctl enable dnsmasq.service

## PiKVM specific additional adjustments

### Delaying startup of dnsmasq

For some reason, on my Raspberry Pi, the interface `eth0` took quite a while to initialize, leading to `dnsmasq` yielding errors on startup since the configured interface was not found at the point in time in the `systemd` initialization when `dnsmasq` started. This means, I had to make sure every interface was fully up until I started `dnsmasq`. For that, I had to modify the systemd service configuration file  `/usr/lib/systemd/system/dnsmasq.service`:

    [Unit]
    [...]
    #After=network.target
    #Before=network-online.target nss-lookup.target
    #Wants=nss-lookup.target
    Requires=network-online.target
    After=network-online.target

### Moving DHCP leases file to tmpfs

Since the filesystem of the PiKVM is read-only by default, we run into errors when `dnsmasq` is trying to save existing DHCP leases to `/var/lib/misc` since this is part of the (read-only) partition `/dev/mmcblk0p2`  mounted at `/`. If one is not interested in having DHCP leases saved, we can move the mentioned directory to the temporary filesystem `tmpfs` (note that every file then is deleted on shutdown). For that, add a new line to `/etc/fstab`:

    [...]
    tmpfs /var/lib/misc      tmpfs  mode=0755               0 0
    [...]

Otherwise one would have to link the directory to one of the existing filesystems and make them writable. 
