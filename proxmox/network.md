## Installing pfsense

1. Configure network interfaces

> cp /etc/network/interfaces /etc/network/interfaces.bak 

> nano /etc/network/interfaces

```
auto lo
iface lo inet loopback
iface lo inet6 loopback

auto enp2s0
iface enp2s0 inet manual
up route add -net xx.xx.xx.ww netmask 255.255.255.224 gw xx.xx.xx.yy dev enp2s0

#Internet
auto vmbr0
iface vmbr0 inet static
    address xx.xx.xx.xx         # public server address
    netmask 255.255.255.224
    gateway xx.xx.xx.yy
    bridge_ports enp2s0
    bridge_stp off
    bridge_fd 0
iface vmbr0 inet6 static
    address  2a01:4f8:202:42dc::2
    netmask  64
    gateway  fe80::1

# WAN
auto vmbr1
iface vmbr1 inet static
    address 10.0.0.1
    netmask 255.255.255.252
    bridge_ports WAN
    bridge_stp off
    bridge_fd 0


# LAN
auto vmbr2
iface vmbr2 inet static
    address 192.168.9.1
    netmask 255.255.255.0
    bridge_ports LAN
    bridge_stp off
    bridge_fd 0
```

2. Download pfsense and ubuntu iso

```
cd /var/lib/vz/template/iso

wget https://frafiles.netgate.com/mirror/downloads/pfSense-CE-2.5.2-RELEASE-amd64.iso.gz

gunzip pfSense-CE-2.5.2-RELEASE-amd64.iso.gz

wget https://releases.ubuntu.com/21.10/ubuntu-21.10-desktop-amd64.iso
```

3. Create pfsense vm

```
Disk size: 8gb
Processor: sockets 1, cores 2
Memory: 512
Bridge: vmbr1
```
Then go in Hardware section and then

Add -> Network device -> vmbr2


4. Run pfsense vm

Install, reboot, then 

Should VLANs be set up now? -> n

WAN interface name -> vtnet0

LAN interface name -> vtnet1

From the main menu press 2 to set up IPs.

WAN:
```
DHCP : no
IPv4 :  
Subnet bit count : 30
Upstream gateway address : 10.0.0.1
DHCP6 : no
IPv6 :
Do you want to revert HTTP as the webConfigurator protocol? no
```
LAN:
```
IPv4 : 192.168.9.254
Subnet bit count : 24
Upstream gateway address :
IPv6 :
DHCP : No
Do you want to revert HTTP as the webConfigurator protocol? no
```

To check it's working:

Select 7 from the main menu and ping 10.0.0.1

5. Configure pfsense vm

You need to either create ssh tunnel to open the pfsense web interface from your machine or run ubuntu vm and visit the address 192.168.9.254 to open the admin panel.

For the ssh tunnel run locally:

> ssh root@IP_PUB_PROXMOX -L 8443:192.168.9.254:443

Then the admin panel will be accessible on https://localhost:8443

6. Optional ubuntu vm installation

Create ubuntu vm with
```
Bridge: vmbr2
```
When installation is complete set up the ip:
```
IPv4 Method = manual
Address = 192.168.9.10
Netmask = 255.255.255.0
Gateway = 192.168.9.254
DNS = 1.1.1.1
```

7. Log in into pfsense admin panel

Either https://localhost:8443/ or https://192.168.9.254

Login: admin

Password: pfsense

In the wizard on step 4 untick the "Block RFC1918 Private Networks" box.

8. Create pfsense route file
```
cat > /root/pfsense-route.sh << EOF
#!/bin/sh
echo 1 > /proc/sys/net/ipv4/ip_forward
ip route change 192.168.9.0/24 via 10.0.0.2 dev vmbr1
EOF
```
Run 
> chmod +x /root/pfsense-route.sh

and then add it for auto-start 
> nano /etc/network/interfaces
```
[...]
#LAN
auto vmbr2
iface vmbr2 inet static
address 192.168.9.1/24
bridge-ports none
bridge-stp off
bridge-fd 0
post-up /root/pfsense-route.sh
```

9. Configure pfsense

Open https://192.168.9.254, then System -> Advanced -> Networking

Tick "Disable hardware checksum offload"

Then go Firewall -> Rules -> Add

```
Action: Pass
Interface: WAN
Protocol: ICMP
```
Apply changes - now you're able to ping 192.168.9.254 from proxmox host again, and all the traffic goes through pfsense.

10. iptables dark fucking magic

> export PUBIP=xx.xx.xx.xx

> export SSHPORT=\<port>


10.1.
```
cat > /root/iptables.sh << EOF
#!/bin/sh
# ---------
# VARIABLES
# ---------
## Proxmox bridge holding Public IP
PrxPubVBR="vmbr0"
## Proxmox bridge on VmWanNET (PFSense WAN side) 
PrxVmWanVBR="vmbr1"
## Proxmox bridge on PrivNET (PFSense LAN side) 
PrxVmPrivVBR="vmbr2"
## Network/Mask of VmWanNET
VmWanNET="10.0.0.0/30"
## Network/Mmask of PrivNET
PrivNET="192.168.9.0/24"
## Network/Mmask of VpnNET
VpnNET="10.2.2.0/24"
## Public IP => Your own public IP address
PublicIP="${PUBIP}"
## Proxmox IP on the same network than PFSense WAN (VmWanNET)
ProxVmWanIP="10.0.0.1"
## Proxmox IP on the same network than VMs
ProxVmPrivIP="192.168.9.1"
## PFSense IP used by the firewall (inside VM)
PfsVmWanIP="10.0.0.2"
EOF
```

10.2.
```
cat >> /root/iptables.sh << EOF
# ---------------------
# CLEAN ALL & DROP IPV6
# ---------------------
### Delete all existing rules.
iptables -F
iptables -t nat -F
iptables -t mangle -F
iptables -X
### This policy does not handle IPv6 traffic except to drop it.
ip6tables -P INPUT DROP
ip6tables -P OUTPUT DROP
ip6tables -P FORWARD DROP
# --------------
# DEFAULT POLICY
# --------------
### Block ALL !
iptables -P OUTPUT DROP
iptables -P INPUT DROP
iptables -P FORWARD DROP
# ------
# CHAINS
# ------
### Creating chains
iptables -N TCP
iptables -N UDP
# UDP = ACCEPT / SEND TO THIS CHAIN
iptables -A INPUT -p udp -m conntrack --ctstate NEW -j UDP
# TCP = ACCEPT / SEND TO THIS CHAIN
iptables -A INPUT -p tcp --syn -m conntrack --ctstate NEW -j TCP
# ------------
# GLOBAL RULES
# ------------
# Allow localhost
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
# Don't break the current/active connections
iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# Allow Ping - Comment this to return timeout to ping request
iptables -A INPUT -p icmp --icmp-type 8 -m conntrack --ctstate NEW -j ACCEPT
EOF
```

10.3.
```
cat >> /root/iptables.sh << EOF
# --------------------
# RULES FOR PrxPubVBR
# --------------------
### INPUT RULES
# ---------------
# Allow SSH server
iptables -A TCP -i \$PrxPubVBR -d \$PublicIP -p tcp --dport ${SSHPORT} -j ACCEPT
# Allow Proxmox WebUI
iptables -A TCP -i \$PrxPubVBR -d \$PublicIP -p tcp --dport 8006 -j ACCEPT
EOF
```

10.4.
```
cat >> /root/iptables.sh << EOF
### OUTPUT RULES
# ---------------
# Allow ping out
iptables -A OUTPUT -p icmp -j ACCEPT
### Proxmox Host as CLIENT
# Allow HTTP/HTTPS
iptables -A OUTPUT -o \$PrxPubVBR -s \$PublicIP -p tcp --dport 80 -j ACCEPT
iptables -A OUTPUT -o \$PrxPubVBR -s \$PublicIP -p tcp --dport 443 -j ACCEPT
# Allow DNS
iptables -A OUTPUT -o \$PrxPubVBR -s \$PublicIP -p udp --dport 53 -j ACCEPT
### Proxmox Host as SERVER
# Allow SSH 
iptables -A OUTPUT -o \$PrxPubVBR -s \$PublicIP -p tcp --sport ${SSHPORT} -j ACCEPT
# Allow PROXMOX WebUI 
iptables -A OUTPUT -o \$PrxPubVBR -s \$PublicIP -p tcp --sport 8006 -j ACCEPT
EOF
```

10.5.
```
cat >> /root/iptables.sh << EOF
### FORWARD RULES
# ----------------
### Redirect (NAT) traffic from internet 
# All tcp to PFSense WAN except ${SSHPORT}, 8006
iptables -A PREROUTING -t nat -i \$PrxPubVBR -p tcp --match multiport ! --dports ${SSHPORT},8006 -j DNAT --to \$PfsVmWanIP
# All udp to PFSense WAN
iptables -A PREROUTING -t nat -i \$PrxPubVBR -p udp -j DNAT --to \$PfsVmWanIP
# Allow request forwarding to PFSense WAN interface
iptables -A FORWARD -i \$PrxPubVBR -d \$PfsVmWanIP -o \$PrxVmWanVBR -p tcp -j ACCEPT
iptables -A FORWARD -i \$PrxPubVBR -d \$PfsVmWanIP -o \$PrxVmWanVBR -p udp -j ACCEPT
# Allow request forwarding from LAN
iptables -A FORWARD -i \$PrxVmWanVBR -s \$VmWanNET -j ACCEPT
EOF
```


10.6.
```
cat >> /root/iptables.sh << EOF
### MASQUERADE MANDATORY
# Allow WAN network (PFSense) to use vmbr0 public adress to go out
iptables -t nat -A POSTROUTING -s \$VmWanNET -o \$PrxPubVBR -j MASQUERADE
EOF
```


> chmod +x /root/iptables.sh

> /root/iptables.sh

The final commands list is in *iptables.sh* in this folder.

### IMPORTANT:
As all the ipv6 traffic is dropped (ip6tables rules from 10.2) we need to disable ipv6 usage in the host system:
```
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.default.disable_ipv6=1
```

11. Nginx setup example

In order to open the server to the internet create any VM, set it up on vmbr2 bridge and assign some ip (i.e. 192.168.9.20). Then:
> sudo apt install nginx

It should be now accessible from the same machine:
> curl localhost

In order for it to be accessible from the global net we need to add a new rule to the firewall. Open pfsense web ui, then Firewall -> NAT -> Port Forward. Add the following rule:
```
Interface : WAN
Protocol : TCP
Destination : Any
Destination Port Range : 8001
Redirect target IP : 192.168.9.20
Redirect target port : 80
Description : Nginx server
```
This will also automatically create a rule in Firewall -> Rules.

Now try to visit http://\<your-public-ip>:8001 - you should see the nginx welcome page.

12. SSH to local virtual machines

In order to be able to SSH in VMs from the hypervisor itself (i.e. `ssh root@192.168.9.20`) do:

Install SSH server on the VM you want to SSH into:
> sudo apt install openssh-server

Add new Firewall -> Rules:
```
Interface : WAN
Protocol : TCP
Source: single host = 10.0.0.1
Destination : LAN net
Destination Port Range : 22
Description : SSH for VMs
```

Still, SSH won't work, in order to see that packets don't even leave the machine:
>  tcpdump -i vmbr1 -p tcp

For also debugging iptables blocked traffic put right after removing all the rules in the rules file (logs are saved to /var/log/syslog):
> iptables -A OUTPUT -p tcp -o vmbr1 -j LOG

Then
```
cat >> /root/iptables.sh << EOF
# --------------------
# RULES FOR PrxVmWanVBR
# --------------------
### Allow being a client for the VMs
iptables -A OUTPUT -o \$PrxVmWanVBR -s \$ProxVmWanIP -p tcp -j ACCEPT
EOF
```
SSH now works from inside of the host machine.
In order to get rid of using VM for accessing pfsense interface it's possible to allow tunneling again. Add another rule:
```
Interface : WAN
Protocol : TCP
Source: single host = 10.0.0.1
Destination : single host = 192.168.9.254
Destination Port Range : 443
Description : Access interface from hypervisor (tunneling)
```
To test the tunnel works again:
> ssh root@IP_PUB_PROXMOX -L 8443:192.168.9.254:443



## Links

https://blog.zwindler.fr/2020/03/09/proxmox-ve-6-pfsense-sur-un-serveur-dedie-2-3/

https://blog.zwindler.fr/2017/07/18/deploiement-de-proxmox-ve-5-sur-un-serveur-dedie-part-2/

https://community.hetzner.com/tutorials/docker-ipv6-securely

https://blog.zwindler.fr/2020/03/02/deploiement-de-proxmox-ve-6-pfsense-sur-un-serveur-dedie/

https://pve.proxmox.com/wiki/Network_Configuration

https://community.hetzner.com/tutorials/install-and-configure-proxmox_ve

https://www.youtube.com/watch?v=25uXpWb4-hE