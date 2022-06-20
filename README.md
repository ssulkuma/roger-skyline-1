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
______________________________________

To create a new user with sudo rights:
```
$ sudo adduser testuser
$ sudo adduser testuser sudo
```
To give it ssh access:
```
$ sudo mkdir /home/testuser/.ssh
$ sudo cp .ssh/id_rsa.pub /home/testuser/.ssh/authorized_keys
```

# Network
For setting up network, from the VirtualBox main window, I go to VM settings -> network and set Adapter 1 to bridged network.

To list all network interfaces, I run the command:
```
$ ifconfig
```
The project asks us to set it a static IP, to do that I run the command:
```
$ sudo vim /etc/netplan/00-installer-config.yaml
```
And write the following in the file:
```
#This file describes the network interfaces available on your system
#For more infromation, see netplan(5).

network:
  ethernets:
    enp0s3:
      dhcp4: no
      dhcp6: no
      addresses: [10.11.254.253/30, ]
      gateway4: 10.11.254.254
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
  version: 2
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

To generate a ssh key (with a passphrase), I run the command:
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
And to disable possible root login and password authentication via ssh, I modify the rules:
```
...
#PermitRootLogin prohibit-password
PermitRootLogin no
...
PasswordAuthentication no
PubkeyAuthentication yes
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

To protect for denial of service attacks (DOS), I want to install fail2ban:
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
```
To take the changes into account, I run the command:
```
$ sudo systemctl restart fail2ban
```
To test if the rule works, I try to ssh from my terminal 3 times with incorrect password and then from vm shell, run the command:
```
$ sudo fail2ban-client status sshd
```
And I should see the banned IP.
______________________________________

To protect from port scanning, I want to install portsentry and nmap (for testing):
```
$ sudo apt install portsentry
$ sudo apt install nmap
```
At first I want to change the portsentry mode to advanced, so I modify the /etc/default/portsentry file:
```
$ sudo vim /etc/default/portsentry
```
```
...
TCP_MODE="atcp"
UDP_MODE="audp"
```
Then I modify the configuration file to have portsentry protect the used open ports & block them from scanning:
```
$ sudo vim /etc/portsentry/portsentry.conf
```
```
...
BLOCK_UDP="1"
BLOCK_TCP="1"
...
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
...
#KILL_HOSTS_DENY="ALL : $TARGET$ : DENY"
....
```
To take changes into account, I restart the service:
```
$ sudo systemctl restart portsentry
```
______________________________________

To list all the running services, I use the command:
```
$ sudo service --status-all
```
The list of services I want to keep running:
```
 [ + ]  apparmor
 [ + ]  apport
 [ + ]  cron
 [ + ]  dbus
 [ + ]  fail2ban
 [ + ]  kmod
 [ + ]  nginx
 [ + ]  portsentry
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
date | tee -a /var/log/update_script.log 
apt-get update | tee -a /var/log/update_script.log 
apt-get upgrade -y | tee -a /var/log/update_script.log
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

# Web Part
For the project we must set up a web server using either Apache or Nginx. I chose Apache. To install it, I run the commands:
```
$ sudo apt update
$ sudo apt install nginx
```
To change the firewall settings of Nginx:
```
$ sudo ufw allow 80/tcp
```
Then by typing in the web browser the address, I check if I got it working.
```
http://10.11.254.253
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

To then create the SSL certificate and fill the information asked, I run the command:
```
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
```
```
Country Name (2 letter code) [AU]:FI
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:Helsinki
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Hive
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:10.11.254.253                       
Email Address []:
```
We also want to negotiate perfect forward secrecy (strong DH group) with clients so we generate the next command:
```
$ sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```
Then I want to create a snippet configuration file to use these changes so I add these lines to it:
```
$ sudo vim /etc/nginx/snippets/self-signed.conf
```
```
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```
To make sure we run the Nginx SSL securely, I modify another snippet conf file to so:
```
$ sudo vim /etc/nginx/snippets/ssl-params.conf
```
```
# from https://cipherli.st/
# and https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html

ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
ssl_ecdh_curve secp384r1;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
# Disable preloading HSTS for now.  You can use the commented out header line that includes
# the "preload" directive if you understand the implications.
#add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;

ssl_dhparam /etc/ssl/certs/dhparam.pem;
```
Finally we want to start using the SSL:
```
$ sudo vim /etc/nginx/sites-available/default
```
```
server {

    # SSL configuration

    listen 10.11.254.253:443 ssl;
    include snippets/self-signed.conf;
    include snippets/ssl-params.conf;
    root /var/www/html;
    index index.html;
}
server {
    listen 10.11.254.253:80;
    server_name 10.11.254.253;
    root /var/www/html;
    index index.html;
    return 301 https://10.11.254.253;
}
```
To allow access to the https, I need to change firewall rules:
```
$ sudo ufw allow 443/tcp
$ sudo ufw status
```
Lastly I want to restart the nginx sevice:
```
$ sudo systemctl restart nginx.service
```
I'll also update the fail2ban with new jail rules against dos attack:
```
$ sudo vim /etc/fail2ban/jail.local
```
```
...
[http-get-dos]
enabled = true
port = http,https
filter = http-get-dos
logpath = /var/log/nginx/acccess.log*
bantime = 1m
findtime = 3m
maxretry = 100

[ban-timeout]
enabled = true
logpath = /var/log/nginx/access.log
filter =
port = 80,443
failregex = ^<ADDR> \S+ \S+(?: \[\])? "[^"]*" 408\s
maxretry = 3
findtime = 3m
bantime = 1m

[port-block]
enabled = true
filter =
failregex = .*\[UFW BLOCK\] IN=.* SRC=<HOST>
logpath = /var/log/syslog
findtime = 3m
maxretry = 10
bantime = 3m
```
```
$ sudo systemctl restart fail2ban
```
______________________________________

For the automated deployment part, I wrote a script. For the script to work, we need to generate a new ssh key without passphrase and name it deploy on another vm for example:
```
$ ssh-keygen
```
Add the key to authorized_file:
```
$ ssh-copy-id -i .ssh/deploy.pub ssulkuma@10.11.254.253 -p 2211
```
The deployment script (make sure paths are correct for local and current):
```
$ vim deploy_script.sh
```
```
#!bin/bash
LOCAL=./web/index.html
DEPLOY=/var/www/html/index.html
DIFF=$(scp -P 2211 -i .ssh/deploy ssulkuma@10.11.254.253:/var/www/html/index.html ./deploy/index.html)
CURRENT=./deploy/index.html
DOT=1
diff $LOCAL $CURRENT

if [ $? -eq 1 ]
then
	echo "Changes in the file identified."
	sleep 2
	echo "Deploying updated version."
	sleep 2
	scp -P 2211 -i .ssh/deploy $LOCAL ssulkuma@10.11.254.253:$DEPLOY
	echo -n "Deploying"
	while [ $DOT -le 5 ]
	do
		sleep 1
		echo -n "."
		DOT=$(($DOT+1))
	done
	echo " Deployment succeeded."
else
	echo "No changes have been made in the file."
fi
```
