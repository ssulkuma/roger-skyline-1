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
For setting up network, from the VirtualBox main window, I go to VM settings -> network and set Adapter 1 to bridged network.

To list all network interfaces, I run the command:
```
$ ifconfig
```

The project asks us to set it a static IP, to do that I run the command:
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
      dhcp4: no
      dhcp6: no
      addresses: [10.11.254.253/30, ]
      gateway4: 10.11.254.254
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```        
After saving the file, I want to apply the changes I made, so I run the command:
```
$ sudo netplan apply
```
To see I've had network connection, I test it with:
```
$ ping google.com
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
$ sudo ufw allow 2211/tcp
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

To list all the running services, I use the command:
```
$ sudo service --status-all
```
The list of services I want to keep running:
```
 [ + ]  apache2
 [ + ]  apparmor
 [ + ]  apport
 [ + ]  atd
 [ + ]  cron
 [ + ]  dbus
 [ + ]  fail2ban
 [ + ]  kmod
 [ + ]  multipath-tools
 [ + ]  postfix
 [ + ]  procps
 [ + ]  rsyslog
 [ + ]  ssh
 [ + ]  udev
 [ + ]  ufw
 ```
 I stop the services I don't need by running the command:
 ```
 $ sudo systemctl disable unattended-upgrades.service 
 ```
______________________________________

For the project, it's required to create a script which updates all the sources of packages and then my packages and logs the information. First I create a new file and write the script to it as follows:
```
$ vim update_script.sh
```
```
#!/bin/bash
date | tee -a /var/log/update_script.log && \
apt update | tee -a /var/log/update_script.log && \
apt upgrade -y | tee -a /var/log/update_script.log
```
To schedule the script to run at the required times, I want to create and modify the crontab:
```
$ sudo crontab -e
```
```
0 4 * * 1 sh /home/ssulkuma/update_script.sh
@reboot sh /home/ssulkuma/update_script.sh
```
______________________________________

It's also required to make a script to monitor changes in /etc/crontab and send email to root if there's been changes. So first, I want to install mailutils:
```
$ sudo apt install mailutils
```
I want to create a mail attachment file and write the needed information there.
```
$ vim crontab_notice.txt
```
```
/etc/crontab has been modified within the past 24 hours.
```
Then I run the command to make a new script:
```
$ vim check_script.sh
```
```
#!/bin/bash
STAT=$(stat -c %Z /etc/crontab)
PREV_STAT=$(cat /var/log/check_script.log)

if [$STAT -eq $PREV_STAT]
then
  echo "/etc/crontab has not been modified within the past 24 hours."
else
  echo "/etc/crontab has been modified within the past 24 hours. An email will be sent to root."
  mail -s "/etc/crontab notification" root@roger < /home/ssulkuma/crontab_notice.txt
fi
stat -c %Z /etc/crontab | tee /var/log/check_script.log
```
To schedule the script to happen at midnight, I change the crontab again and add the line to it:
```
$ sudo crontab -e
```
```
0 0 * * * sh /home/ssulkuma/check_script.sh
```
With the mailutils, we also need to update a new rule for firewall, so I check the info on Postfix and add a new port rule:
```
$ sudo ufw app list
$ sudo ufw app info Postfix
$ sudo ufw allow 25/tcp
```
# Web Part
For the project we must set up a web server using either Apache or Nginx. I chose Apache. To install it, I run the commands:
```
$ sudo apt update
$ sudo apt install apache2
```
To change the firewall settings of Apache:
```
$ sudo ufw app list
$ sudo ufw app info Apache Full
$ sudo ufw allow 80/tcp
```
Then by typing in the web browser the address, I check if I got it working.
```
http://192.168.56.1
```
To change the html of the page I modify the file:
```
$ vim /var/www/html/index.html
```
```
<html>
<head>
<style>
body {
	background-color: black;
	font-family: Verdana;
	font-size: 12px;
	color: white;
	margin-left: 150px;
	margin-top: 50px;
}

h1 {
	color: purple;
	font-size: 14px;
}

a {
	color: white;
	font-size: 11px;
	text-transform: uppercase;
	text-decoration: none;
	letter-spacing: 2px;
	transition: all .2s ease-in;
}

a:hover {
	color: purple;
	letter-spacing: 0.6em;
}
input {
	width: 180px;
	height: 14px;
	background-color: black;
	color: white;
	border: 0;
	border-bottom: 1px solid gray;
}
input:focus {
	outline: none;
}

.login {
	margin-left: 130px;
}
</style>
	<title>Roger Skyline 1</title>
</head>
<body>
	<h1>Roger Skyline 1</h1>
	<p>[ Username ] <input></input></br>
	[ Password&nbsp; ] <input type="password"></input></p>
	<div class="login"><a href="https://www.youtube.com/watch?v=ZVuToMilP0A">Login</a></div>
</body>
</html>
```
______________________________________

For the website, we're going to create a self-signed SSL. To do that I first enable mod_ssl (an Apache module that provides support for SSL encryption) and then restart the service to take these changes into account:
```
$ sudo a2enmod ssl
$ sudo systemctl restart apache2
```
To then create the certificate and fill the information asked, I run the command:
```
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```
```
Country Name (2 letter code) [AU]:FI
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:Helsinki
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Hive
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:10.11.254.253                       
Email Address []:ssulkuma@roger.hive.fi
```
Then I want to modify the configuration file to use these changes so I add these lines to it:
```
$ sudo vim /etc/apache2/sites-available/000-default.conf
```
```
<VirtualHost *:443>
	ServerName 192.168.56.1
	SSLEngine on
	SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
	SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
</VirtualHost>
<VirtualHost *:80>
	ServerName 10.11.254.253
	Redirect / https://10.11.254.253
</VirtualHost>
```
To enable the configuration file and test that the syntax is okay with it, I run the commands:
```
$ sudo a2ensite
$ sudo apache2ctl configtest
```
To allow access to the https, I need to change firewall rules:
```
$ sudo ufw allow 443/tcp
$ sudo ufw status
```
Then from the main window of the VM, I go to Settings -> Network -> Port forwarding and add a new rule (https / - / 443 / - / 443). Lastly I want to restart the apache sevice:
```
$ sudo systemctl restart apache2
```
I'll also update the fail2ban with new jail rules for apache:
```
$ sudo vim /etc/fail2ban/jail.local
```
```
...
[apache]
enabled = true
port = http,https
bantime = 1m
findtime = 1m
maxretry = 3

[apache-overflows]
enabled = true
port = http,https
bantime = 1m
findtime = 1m
maxretry = 3
```
```
$ sudo systemctl restart fail2ban
```
______________________________________
