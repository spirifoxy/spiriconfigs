## Proxmox installation on Hetzner dedicated server

1. Rescue mode

Open https://robot.your-server.de/server.

Main functions > Servers > Server Label > Rescue

Linux > 64 bit > Activate rescue system

2. Create server config

ssh root@yourserverip

nano install-config.txt

```
DRIVE1 /dev/sda
DRIVE2 /dev/sdb

SWRAID 1
SWRAIDLEVEL 0 # Use 1 for Raid 1

BOOTLOADER grub
HOSTNAME yourhostname.example

PART /boot ext3 512M
PART lvm vg0 all

LV vg0 root / ext4 50G
LV vg0 swap swap swap 8G
LV vg0 var /var  ext4  300G

IMAGE /root/.oldroot/nfs/install/../images/Debian-1010-buster-64-minimal.tar.gz
```

3. Debian installation

installimage -a -c install-config.txt

Reboot and check the partition scheme:
```
lsblk
...
pvs
...
vgs
...
lvs
...
```

Check the OS release info:
```
cat /etc/os-release
```

Optionally adding more space to a logical volume:
```
lvextend -r -L +20G /dev/vg0/<lvname>
lvextend -r -l +100%FREE /dev/vg0/<lvname> #add all free to this lv
```

Disable ssh using password:
```
nano /etc/ssh/sshd_config
```
Then make the following changes in the file:
```
PubkeyAuthentication yes
PasswordAuthentication no
```

4. Proxmox installation

Add Proxmox VE repository
```
echo "deb http://download.proxmox.com/debian/pve buster pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
```

Add Proxmox VE repository key
```
curl -#o /etc/apt/trusted.gpg.d/proxmox-ve-release-6.x.gpg http://download.proxmox.com/debian/proxmox-ve-release-6.x.gpg
```

Update the system:
```
apt update -y && apt full-upgrade -y && apt dist-upgrade -y && apt autoremove -y

reboot
```

Remove the packages not needed as Proxmox will bring its own version
```
apt purge firmware-bnx2x firmware-realtek firmware-linux firmware-linux-free firmware-linux-nonfree
```

Install Proxmox VE packages
```
apt install proxmox-ve postfix open-iscsi

apt remove os-prober

rm /etc/apt/sources.list.d/pve-enterprise.list && apt-get update

reboot
```
5. Set up root password
```
passwd root
```


## Links

https://community.hetzner.com/tutorials/hyperconverged-proxmox-cloud

https://community.hetzner.com/tutorials/install-and-configure-proxmox_ve

https://computingforgeeks.com/install-proxmox-ve-on-hetzner-root-server/

https://www.indivar.com/blog/how-to-setup-proxmox-on-hetzner-dedicated-server/#hostname