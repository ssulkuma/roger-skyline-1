# roger-skyline-1
Basics about system and network administration school project of Hive Helsinki. 

In this project you have to run a Virtual Machine (VM) with the Linux OS of your choice in the hypervisor of your choice. I chose to deploy Ubuntu Server 20.04. LTS on VirtualBox. For the VM, disk size of 8GB and with at least one 4.2GB partition are required.

# Installation
From the main window of VirtualBox, I create a new VM with the operating system of my choice (Linux / Ubuntu 64-bit) and give it the disk size of 8GB as required. Adding the iso-file from the settings -> storage, I then follow the installator options of the Ubuntu Server. I set the 4.2GB partition at the Storage part of the installation and allow OpenSSH service, otherwise default settings are used. Lastly, I give it a login, password and a server name and let it finish installation.

After logging in, updates / upgrades are available so I run the commands:
```
$ sudo apt update
$ sudo apt upgrade
```

Making sure that disk size and partition has been set correctly, I run the command:
```
$ lsblk | grep sda
```

# Network
For setting up network, from the VirtualBox main window, I go to Global tools -> Host Network Manager -> create a new host-only network (vboxnet0 / 192.168.56.1/24 / disable DHCP Server). Then, from VM settings -> network, I set Adapter 1 to NAT and Adapter 2 to host-only network / vboxnet0.

To list all network interfaces, I run the command:
```
$ ifconfig
```

I see the enp0s3 (NAT) has an IP set, but enp0s8 (host-only network) does not. The project asks us to set it a static IP, to do that I run the command:
```
$ sudo vim /etc/netplan/01-netcfg.yaml
```

And write the following in the file:
```
#This file describes the network interfaces available on your system
#For more infromation, see netplan(5).

network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: yes
    enp0s8:
      dhcp4: no
      dhcp6: no
      addresses: [192.168.56.253/30, ]
      gateway4: 192.168.56.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```        
After saving the file, I want to apply the changes I made, so I run the command:
```
$ sudo netplan apply
```

Check wether I've succeeded:
```
$ ifconfig
```
______________________________________

To generate a ssh key, I run the command:
```
$ ssh-keygen
```

To copy the public key to server, I run the command:
```
$ ssh-copy-id -i .ssh/id_rsa.pub user@host
```

For the ssh the project requires changing the port. So I make changes in the /etc/ssh/sshd_config file:
```
$ sudo vim /etc/ssh/sshd_config
```
I find the line with #Port 22 from the file and add my new port on the line under it (without commenting it). This will override the default port. The part of file should look like this:
```
...
#Port 22
Port 2211
...
```

To take the changes into account, I'll restart the ssh service with the command:
```
$ sudo systemctl restart ssh.service
```
______________________________________

For the project, we're required to set firewall rules for services used outside our VM. In this case, the ssh service. First I want to check the status of my firewall (uwf = uncomplicated firewall):
```
$ sudo ufw status
```
The firewall's status is inactive, before activating it I want to change access for IPv6. To do that, I run the command:
```
$ sudo vim /etc/default/ufw
```
And change IPv6 part to:
```
IPV6=no
```
Now I want to activate the firewall, I'll do that by running the command:
```
$ sudo ufw enable
```
If I want to check existing rules for applications, I can run the command:
```
$ sudo ufw app list
```
To change the firewall rules:
```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo allow 2211/tcp
```
______________________________________

To protect for denial of service attacks (DOS) and protection for port scanning, I want to install fail2ban:
```
$ sudo apt-get install fail2ban
```
To enable fail2ban, I run the command:
```
$ sudo systemctl start fail2ban
```
To set up the configuration, I want to create /etc/fail2ban/jail.local file instead of using the /etc/fail2ban/jail.conf and modify that file as follows:
```
$ sudo vim /etc/fail2ban/jail.local
```
```
[sshd]
enabled = true
port = 2211
bantime = 1m
findtime = 1m
maxretry = 3

[portscan]
enabled = true
bantime = 1m
findtime = 1m
maxretry = 1
```
To take the changes into account, I run the command:
```
$ sudo systemctl restart fail2ban
```
______________________________________
