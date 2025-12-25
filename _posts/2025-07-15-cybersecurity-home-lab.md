---
title: Setting Up a Home Lab/Offensive Infrastructure Using Your Unused PC
date: 2025-12-25 +/-TTTT
categories: [REDTEAM]
tags: [lab,ludus]
---

# Setting Up a Home Lab/Offensive Infrastructure Using Your Unused PC

# This page is not finished yet.

---

Lately, Ludus has gained significant attention in the cybersecurity community due to its powerful features and capabilities. In this blog, we'll walk through the process of setting up a home lab or offensive infrastructure using an unused PC. This setup will allow you to practice various cybersecurity techniques without relying on expensive subscriptions.

If you're a cybersecurity professional looking for a network to practice on, this is the most cost-effective solution available. Ludus comes with its own set of predefined labs like GOAD, SCCM lab, and more.

You can reuse the same home infrastructure and simply switch VPS/domains for different offensive operations.

## Requirements

~$2-5 per month for a single 512MB - 2GB RAM VPS (Consider looking beyond expensive countries for affordable options)
A computer or laptop with sufficient RAM and vCPU cores (SSD is highly recommended, and an NVIDIA GPU can be used for hash cracking)

## What You'll Achieve

In this blog post, we'll cover:

- Proxmox + Ludus setup
- WireGuard reverse VPN from VPS
- The magic of socat UDP and SSH tunneling
- Hash cracking station with a discrete GPU
- BONUS: Domain registration, SSL, and DNS setup for Havoc C2

By following this guide, you will be able to:

- Host infrastructure locally and expose it to the internet without incurring high costs.
- Access your home lab via VPN from anywhere, which is ideal for collaboration.
- Perform hash cracking on the same Proxmox node.
- Keep all your offensive tools organized and easily accessible.

## DISCLAIMER - READ THIS CAREFULLY!

Please note that the information provided in this blog is intended solely for educational and research purposes. Unauthorized use of this knowledge to attack networks beyond your authorized scope may result in legal consequences. Additionally, improper setup could potentially expose your home network to the public internet, so exercise extreme caution.

---

## Part 1 - Installing Debian, Proxmox, and Ludus

Start by downloading Debian from https://www.debian.org/distrib/. Once downloaded, flash it onto an SD card and follow the installation instructions. After completing the installation, proceed to install Ludus following the guide available at https://docs.ludus.cloud/docs/quick-start/install-ludus/ .

### /etc/pve/storage.cfg config

```c
root@proxmox:~# cat /etc/pve/storage.cfg
dir: local
    path /var/lib/vz
    content images,rootdir
    prune-backups keep-all=1

dir: HDD
    path /mnt/HDD
    content backup,iso,vztmpl,snippets
    prune-backups keep-all=1
```

### Ludus range

```
export LUDUS_API_KEY='INFR.redacted'
```



## Part 2 - VPS, WireGuard, and Firewall Rules

### Wireguard configuration


```bash
wg-quick up ~/bridge.conf

sudo wg-quick up bridge

sudo wg show

sudo wg-quick down bridge
```

Make sure to update the keys, network interface names, and 10.8.8.0/24 CIDR in all configuration files to match your specific setup.


**PROXMOX:**

```yaml
root@proxmox:~# cat /etc/wireguard/bridge.conf
[Interface]
Address = 10.10.10.3/32
PrivateKey = redacted
ListenPort = 60066
                                                                                                                                                                                # PostUp: Set up forwarding rules, NAT, and routing
PostUp = iptables -A FORWARD -i bridge -j ACCEPT
PostUp = iptables -A FORWARD -o bridge -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o enp4s0 -j MASQUERADE
PostUp = iptables -t nat -A POSTROUTING -s 10.10.10.1 -o vmbr1000 -j MASQUERADE
PostUp = iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -d 10.8.8.0/24 -j MASQUERADE
PostUp = ip route del 10.8.8.0/24 via 192.0.2.108 dev vmbr1000 2>/dev/null || true
PostUp = ip route add 10.8.8.0/24 via 192.0.2.108 dev vmbr1000

# PostDown: Clean up rules and routing (reverse order)
PostDown = ip route del 10.8.8.0/24 via 192.0.2.108 dev vmbr1000 2>/dev/null || true
PostDown = iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -d 10.8.8.0/24 -j MASQUERADE 2>/dev/null || true
PostDown = iptables -t nat -D POSTROUTING -s 10.10.10.1 -o vmbr1000 -j MASQUERADE 2>/dev/null || true
PostDown = iptables -t nat -D POSTROUTING -o enp4s0 -j MASQUERADE 2>/dev/null || true
PostDown = iptables -D FORWARD -o bridge -j ACCEPT 2>/dev/null || true
PostDown = iptables -D FORWARD -i bridge -j ACCEPT 2>/dev/null || true

[Peer]
PublicKey = redacted
AllowedIPs = 10.10.10.0/24
Endpoint = 94.141.102.251:51820
PersistentKeepalive = 25
root@proxmox:~#
```

**Bridge on VPS:**

```yaml
root@878172:~# cat /etc/wireguard/wg0.conf
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = redacted
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE; ip route add 10.8.8.0/24 via 10.10.10.3 dev wg0
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE; ip route del 10.8.8.0/24 via 10.10.10.3 dev wg0

# Router to networks (bridge interface on proxmox)
[Peer]
PublicKey = redacted
AllowedIPs = 10.8.0.0/16, 10.10.10.3/32
Endpoint = 192.168.1.123:60066
PersistentKeepalive = 25

# Client 1
[Peer]
PublicKey = redacted
AllowedIPs = 10.10.10.5/32
root@878172:~#
```


**Client 1 VPN**

```yaml
[Interface]
PrivateKey = redacted
Address = 10.10.10.5/32
DNS = 1.1.1.1, 8.8.8.8

[Peer]
PublicKey = redacted
AllowedIPs = 10.8.0.0/16, 10.10.10.0/24
Endpoint = 94.141.102.251:51820
PersistentKeepalive = 25
```







## Part 3 - Socat UDP and SSH Tunneling







## Part 4 - Hash Cracking Station with NVIDIA Discrete GPU

### Guest OS (kali)

```bash
sudo nano /etc/default/grub
sudo update-grub
```

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt vfio-pci ids=10de:2504,10de:228e"
```

```bash
sudo apt install nvidia-driver nvidia-cuda-toolkit nvidia-settings
sudo apt install nvidia-cuda-toolkit
sudo apt install nvidia-settings
sudo nvidia-smi
lspci -k -nn | grep -A 2 -i nvidia
sudo modprobe nvidia
sudo dmesg| grep -i nvidia
sudo hashcat -I


sudo apt-get purge nvidia*
sudo apt-get autoremove

sudo apt install xrdp -y
sudo systemctl start xrdp -y
sudo systemctl start xrdp
sudo systemctl enable xrdp
```

https://developer.download.nvidia.com/compute/nvidia-driver/redist/nvidia_driver/linux-x86_64/nvidia_driver-linux-x86_64-535.216.03-archive.tar.xz

https://us.download.nvidia.com/XFree86/Linux-x86_64/570.144/NVIDIA-Linux-x86_64-570.144.run

```bash
lspci -nnk | grep -A 3 -i nvidia
dpkg -l | grep nvidia
sudo modprobe nvidia
dmesg | grep nvidia
lspci -nnk | grep -A 3 -i vfio
sudo dkms build -m nvidia -v 535.216.03
sudo dkms install -m nvidia -v 535.216.03
dmesg | grep -i nvidia
sudo apt install linux-headers-6.11.2-amd64
sudo dkms build -m nvidia -v 535.216.03
sudo dkms install -m nvidia -v 535.216.03
sudo apt remove --purge nvidia-*
sudo apt update -y
sudo apt autoremove -y
sudo apt install nvidia-driver
sudo apt install linux-headers-$(uname -r)
sudo dkms build -m nvidia -v 535.216.03
sudo dkms install -m nvidia -v 535.216.03
sudo modprobe nvidia
nvidia-smi
lsmod | grep nvidia
```

### Host (Proxmox)

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction nofb nomodeset video=vesafb:off,efifb:off"
```

```
/etc/modprobe.d/blacklist.conf
```

```
blacklist nouveau snd_hda_intel
options nouveau modeset=0

blacklist radeon
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
options vfio_iommu_type1 allow_unsafe_interrupts=1
options kvm ignore_msrs=1
blacklist nvidiafb
options vfio-pci ids=10de:2504,10de:228e disable_vga=1
```

```
/etc/pve/qemu-server/100.conf
```

```
boot: order=scsi0;net0
cores: 2
cpu: x86-64-v2-AES
hostpci0: mapping=GPU
memory: 8192
meta: creation-qemu=9.0.2,ctime=1737633367
name: kali
net0: virtio=BC:24:11:76:D7:7F,bridge=vmbr0,firewall=1
numa: 0
ostype: l26
scsi0: HDD:vm-100-disk-0,iothread=1,size=128G
scsihw: virtio-scsi-single
smbios1: uuid=d4c82d4b-194f-45af-9a4e-10c621533089
sockets: 2
vmgenid: 873207c3-c88c-4146-88e1-5af464242e64
```

### Sources

```
deb http://ftp.pl.debian.org/debian bookworm main contrib
deb http://ftp.pl.debian.org/debian bookworm-updates main contrib
deb http://security.debian.org bookworm-security main contrib

deb [arch=amd64] https://apt.releases.hashicorp.com bookworm main
deb http://download.proxmox.com/debian/ceph-quincy bookworm no-subscription
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```





## Part 5 - Setting Up a Redirector: Domain, SSL, and Havoc DNS Listener

Service

```yaml
[Unit]
Description=SSH Port Forwarding for apache web server
After=network.target

[Service]
ExecStart=/usr/bin/ssh -N -L 0.0.0.0:80:127.0.0.1:80 -i /root/.ssh/id_rsa proxmox@192.168.1.123
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```