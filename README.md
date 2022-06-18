## Install Proxmox on Hetzner Dedicated (iso mode with UEFI)
[![Build Status](https://files.ariadata.co/file/ariadata_logo.png)](https://ariadata.co)

![](https://img.shields.io/github/stars/ariadata/proxmox-hetzner.svg)
![](https://img.shields.io/github/watchers/ariadata/proxmox-hetzner.svg)
![](https://img.shields.io/github/forks/ariadata/proxmox-hetzner.svg)
---
### Assume that our servers info is :
  * My Interface : `enp7s0`
  	* `(udevadm info -e | grep -m1 -A 20 ^P.*eth0 | grep ID_NET_NAME_PATH | cut -d'=' -f2)`
  * Main IP4 and Netmask : `148.251.235.75/27`
  	* `(ip address show "$(udevadm info -e | grep -m1 -A 20 ^P.*eth0 | grep ID_NET_NAME_PATH | cut -d'=' -f2)" | grep global | grep "inet "| xargs | cut -d" " -f2)`
  * Main IP4 Gateway : `148.251.235.65`
  	* `(ip route | grep default | xargs | cut -d" " -f3)`
  * MAC address : `a8:a1:59:55:3b:43`
  	* `(ifconfig eth0 | grep -o -E '([[:xdigit:]]{1,2}:){5}[[:xdigit:]]{1,2}')`
  * IPv6 CIDR : `2a01:4f8:201:3315::2/64`
  	* `(ip address show "$(udevadm info -e | grep -m1 -A 20 ^P.*eth0 | grep ID_NET_NAME_PATH | cut -d'=' -f2)" | grep global | grep "inet6 "| xargs | cut -d" " -f2)`
  * Public Subnet CIDR: `46.40.125.209/28`
  	* Get this from robot
  * Private subnet : `192.168.20.0/24`
  * VLAN ID : `4000`
  * private VLAN CIDR (for can comunicate with hetzner cloud) : `10.0.1.5/24`
---

#### Prepare the rescue from hetzner robot :
* Select the Rescue tab for the specific server, via the hetzner robot manager
* * Operating system=Linux
* * Architecture=64 bit
* * Public key=*optional*
* --> Activate rescue system
* Select the Reset tab for the specific server,
* Check: Execute an automatic hardware reset
* --> Send
* Wait a few mins
* Connect via ssh/terminal to the rescue system running on your server

#### Install requirements and Install Proxmox:
```shell
apt -y install ovmf wget 
wget -O pve.iso http://download.proxmox.com/iso/proxmox-ve_7.2-1.iso
```
* For initial proxmox installer via `VNC` :
```shell
printf "change vnc password\n%s\n" "abcd_123456" | qemu-system-x86_64 -enable-kvm -bios /usr/share/ovmf/OVMF.fd -cpu host -smp 4 -m 4096 -boot d -cdrom ./pve.iso -drive file=/dev/nvme0n1,format=raw,media=disk,if=virtio -drive file=/dev/nvme1n1,format=raw,media=disk,if=virtio -vnc :0,password -monitor stdio -no-reboot
```
* Connect with `VNC client` to `148.251.235.75` with password `abcd_123456`

* Install Proxmox and attention to these :
  * choose `zfs` partition type
  * choose `lz4` in compress type of advanced partitioning
  * do not add real IP info in network configuration part (just leave defaults!)
  * close VNC window after system rebooted and waits for reconnect


* Run this command to bring up new installed proxmox in port `5555`
```shell
qemu-system-x86_64 -enable-kvm -bios /usr/share/ovmf/OVMF.fd -cpu host -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22 -smp 4 -m 4096 -drive file=/dev/nvme0n1,format=raw,media=disk,if=virtio -drive file=/dev/nvme1n1,format=raw,media=disk,if=virtio
```
* Login via SSH or ([WinSCP](https://winscp.net/eng/download.php)) To `148.251.235.75` with port `5555` with password that you entered during install.

#### Edit `/etc/network/interfaces` file due to your requirements : 
* Use this template for basic interface. (**change parameters manually**)
* For `Main IP` replace these lines to contents of file  :
* for `Main vmbr0` you can use automatic creation with this command :
```sh
bash <(curl -sSL https://github.com/ariadata/proxmox-hetzner/raw/main/files/update_main_vmbr0_basic_from_template.sh)
```
Or Continue with manual way : 

```apacheconf
# network interface settings; autogenerated
# Please do NOT modify this file directly, unless you know what
# you're doing.
#
# If you want to manage parts of the network configuration manually,
# please utilize the 'source' or 'source-directory' directives to do
# so.
# PVE will preserve these directives, but will NOT read its network
# configuration from sourced files, so do not attempt to move any of
# the PVE managed interfaces into external files!

source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

iface lo inet6 loopback

iface enp7s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 148.251.235.75/27
    gateway 148.251.235.65
    bridge-ports enp7s0
    bridge-stp off
    bridge-fd 1
    bridge-vlan-aware yes
    bridge-vids 2-4094
    hwaddress a8:a1:59:55:3b:43
    pointopoint 148.251.235.65
    up sysctl -p

iface vmbr0 inet6 static
    address 2a01:4f8:201:3315::2/64
    gateway fe80::1
```

* For `private subnet` append these lines to interface file  :
```apacheconf
auto vmbr1
iface vmbr1 inet static
    address 192.168.20.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up   iptables -t nat -A POSTROUTING -s '192.168.20.0/24' -o vmbr0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s '192.168.20.0/24' -o vmbr0 -j MASQUERADE
    post-up   iptables -t raw -I PREROUTING -i fwbr+ -j CT --zone 1
    post-down iptables -t raw -D PREROUTING -i fwbr+ -j CT --zone 1

iface vmbr1 inet6 static
	address 2a01:4f8:201:3315:1::1/80
```

* For `public subnet` append these lines to interface file  :
```apacheconf
auto vmbr2
iface vmbr2 inet static
    address 46.40.125.209/28
    bridge-ports none
    bridge-stp off
    bridge-fd 0

iface vmbr2 inet6 static
    address 2a01:4f8:201:3315:2::1/80
```

* For `vlan support` append these lines to interface file  :
  * You have to create a vswitch with ID `4000` in your robot panel of hetzner. 
```apacheconf
auto vlan4000
iface vlan4000 inet static
    address 10.0.1.5/24
    mtu 1400
    vlan-raw-device vmbr0
    up ip route add 10.0.0.0/16 via 10.0.1.1 dev vlan4000
    down ip route del 10.0.0.0/16 via 10.0.1.1 dev vlan4000
```

* Poweroff `ssh with port 5555`: 
```shell
poweroff
```

* Reboot main `rescue` ssh :
```shell
reboot
```

* after a few minutes , login again to your proxmox server with ssh on port `22`

### Post Install : 
* Change ssh port , root password, and relogin with new port agian :
```shell
bash <(curl -Ls https://gist.github.com/pcmehrdad/2fbc9651a6cff249f0576b784fdadef0/raw)
passwd
```

* Config hostname,timezone and resolv file :
```shell
hostnamectl set-hostname proxmox-example
timedatectl set-timezone Europe/Istanbul
printf "nameserver 1.1.1.1\nnameserver 2606:4700:4700::1111\n" > /etc/resolv.conf
```
* edit `/etc/hosts` file like this :
```apacheconf
127.0.0.1 localhost.localdomain localhost
148.251.235.75 proxmox-example
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
2a01:4f8:201:3315::2 proxmox-example
```

* run this commands:
```shell
systemctl disable --now rpcbind rpcbind.socket

sed -i 's/^\([^#].*\)/# \1/g' /etc/apt/sources.list.d/pve-enterprise.list

echo "deb [arch=amd64] http://download.proxmox.com/debian/pve bullseye pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription-repo.list

sed -i "s|ftp.*.debian.org|ftp.debian.org|g" /etc/apt/sources.list

apt update && apt -y upgrade && apt -y autoremove

pveupgrade

sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service

apt install -y libguestfs-tools unzip iptables-persistent

# apt install net-tools

echo "nf_conntrack" >> /etc/modules
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/99-proxmox.conf
echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.d/99-proxmox.conf
echo "net.netfilter.nf_conntrack_max=1048576" >> /etc/sysctl.d/99-proxmox.conf
echo "net.netfilter.nf_conntrack_tcp_timeout_established=28800" >> /etc/sysctl.d/99-proxmox.conf

```

* Limit ZFS Memory Usage According to [This Link](https://pve.proxmox.com/wiki/ZFS_on_Linux#sysadmin_zfs_limit_memory_usage) :
```shell
echo "options zfs zfs_arc_min=$[6 * 1024*1024*1024]" >> /etc/modprobe.d/99-zfs.conf
echo "options zfs zfs_arc_max=$[12 * 1024*1024*1024]" >> /etc/modprobe.d/99-zfs.conf
update-initramfs -u
```

* Update and `reboot` your system!
```shell
apt update && apt -y upgrade && apt -y autoremove
reboot
```
#### Login to `Web GUI`:
https://IP_ADDRESS:8006/
#### Do other configs!:
> add MASQUERADE and NAT rules, by using sample [iptables-samplefile](https://github.com/ariadata/proxmox-hetzner/raw/main/files/iptables-sample)
```bash
iptables -t nat -A PREROUTING -d 1234/32 -p tcp --dport 10001 -j DNAT --to 192.168.20.100:22
iptables -t nat -A PREROUTING -d 1.2.3.4/32 -p tcp -m multiport --dports 80,443,8181 -j DNAT --to-destination 192.168.1.2
```

> install `Webmin` and configure raid,firewalld,Log-rotate with [webmin on debian 11](https://www.howtoforge.com/how-to-install-webmin-on-debian-11/)

#### Useful Links :
```
https://github.com/extremeshok/xshok-proxmox
https://github.com/extremeshok/xshok-proxmox/tree/master/hetzner
https://88plug.com/linux/what-to-do-after-you-install-proxmox/
https://gist.github.com/gushmazuko/9208438b7be6ac4e6476529385047bbb
https://github.com/johnknott/proxmox-hetzner-autoconfigure
https://github.com/CasCas2/proxmox-hetzner
https://github.com/west17m/hetzner-proxmox
https://github.com/SOlangsam/hetzner-proxmox-nat
https://github.com/HoleInTheSeat/ProxmoxStater
https://github.com/rloyaute/proxmox-iptables-hetzner
```

[Useful Helpers](https://tteck.github.io/Proxmox/)

[firewalld-cmd](https://computingforgeeks.com/how-to-install-and-configure-firewalld-on-debian/)

[proxmox-setup on blog](https://mehrdad.ariadata.co/notes/proxmox-setup-network-on-hetzner/)
