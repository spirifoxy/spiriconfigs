## Adding fail2ban config

1. install fail2ban

```
apt-get install fail2ban
```

2. use /etc/fail2ban/jail.conf as a template for configuration

```
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

nano /etc/fail2ban/jail.local
```

3. add the following settings at the bottom of the jail.local
```
[proxmox]
enabled = true
port = https,http,8006
filter = proxmox
logpath = /var/log/daemon.log
maxretry = 3
# 1 hour
bantime = 3600
```

Optionally change default settings and ssh port:
```
[DEFAULT]
bantime = 84600
findtime = 600
maxretry = 3
destemail = root@example.com
sendername = Fail2ban
action = %(action_mwl)s
[sshd]
enabled = true
port = <port>
```

4. create conf for proxmox in fail2ban

```
nano /etc/fail2ban/filter.d/proxmox.conf 
```

```
[Definition]
failregex = pvedaemon\[.*authentication failure; rhost=<HOST> user=.* msg=.*
ignoreregex =
```

See if it's working:
```
fail2ban-regex /var/log/daemon.log /etc/fail2ban/filter.d/proxmox.conf
```

5. Final steps

```
systemctl restart fail2ban
```

View currently banned IPs:
```
zgrep 'Ban' /var/log/fail2ban.log*

zgrep 'Ban' /var/log/fail2ban.log* | wc -l
```

Unban yourself:

```
fail2ban-client unban --all
```

## Links

https://pve.proxmox.com/wiki/Fail2ban

https://www.indivar.com/blog/how-to-setup-proxmox-on-hetzner-dedicated-server