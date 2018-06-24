# pisql-server
## setup a mysql server on a raspberry pi with some security

### introduction
The purpose of this exercise is to guide you through the process of setting up a MySQL server on a RPI and connecting to it through a SSH tunnel on your local network.  The server and client are both linux distributions but I do have a few notes about using a Windows client.  I am not a linux wiz so I'm not sure if there would be any difference with the default SSH configuration using a different server distribution.  At the end there is a bit on adding more security which is not really required on your local network but good to learn for servers on the internet like DigitalOcean.

### make a bootable rasbian sd
1. I did this exercise on a 2 GB sd card and it worked but it's close.  Get 4 gb or greater if you are buying a new card.
1. Download [Rasbian](https://www.raspberrypi.org/downloads/raspbian/).  This is a RPI specific lnius distribution based on Debian.  The lite version does not include a desktop environment (GUI) which is fine for this exercise.  If you want the dsktop environment get the version with desktop but make sure your sd card is at least 8 GB.
1. I like [Etcher](https://etcher.io/) for buring bootable media.

### configue remote access
a. boot up pi with monitor and keyboard
b. login with default credentials - pi/raspberry (root login is not permitted by default)
c. $ sudo raspi-config (enter for select and escape for back)
d. change password of 'pi' user
e. network options -> hostname (i use the hostname serverpi)
f. network options -> wi-fi (not required if wired connection is being use, which is a good idea if possible)
g. interfacing options -> ssh -> enable
h. esc to exit config menu
i. $ ip addr show (ip will not be needed because of hostname but good idea to grab ip now)

### login though ssh
a. *Ubuntu* $ ssh pi@serverpi (there will be a warning about this being the first time connecting to computer so make sure you know what it is.)
b. *Windows* need to have putty.  ip address will work in putty.  pi should keep same ip if router is not re-started.  might keep same address after restart too.  hostname might work in putty but i have not tried

### install and configure mysql 
a. ssh into serverpi
b. $ sudo apt-get install mysql-server
c. $ sudo mysql_secure_installation (default root password is blank.  no not chnage root password and say yes to all other options)
d. $ sudo mysql (open mysql shell as root user.  shell will say MariaDB which is almost the same as mysql)
e. GRANT ALL PRIVILEGES ON *.* TO 'username'@'localhost' IDENTIFIED BY 'password'; (create a new user will full privliges)
f. exit; (exits mysql)
g. $ mysql -u username -p (mysql shell can now be opened without root)

### use mysql through ssh tunnel from Ubuntu client
a. $ ssh pi@serverpi -L 3307:127.0.0.1:3306 -N (use pi user password.  this will open the tunnel.  if the bash shell is closed the tunnel will also close.  usering port 3307 instead of default 3306 incase ther is a local mysql server running on client)
b. $ mysql --host=127.0.0.1 --port=3307 -u username -p (will open mysql shell.  use mysql username password.  this will require a mysql client be installed [$ sudo apt install mariadb-client])
c. $ mysql --host=127.0.0.1 --port=3307 -u username -p < testdb_schema.sql (script needs to start with 'USE db_name')
d. $ mysql --host=127.0.0.1 --port=3307 -u username -p db_name < script_to_run.sql (script will run on db_name)
e. *Windows* here are putty instructions for the tunnel https://www.linode.com/docs/databases/mysql/create-an-ssh-tunnel-for-mysql-remote-access/

### add some security
a. ssh into server pi
b. $ sudo groupadd mysqlusers (create group for mysql users)
c. $ sudo useradd -g mysqlusers testuser (crate testuser in mysqlusers)
d. $ sudo passwd testuser (create password for new user)
e. $ sudo userdel -r testuser (in the future this will let you delete user + home dirrectory and mail spool)
f. $ sudo nano /etc/ssh/sshd_config (add the text below to sshd_config.  this will control access by groups and then resrict assess of mysqlusers group.  last line removes shell access from ssh.  adding pi group preserves ssh access for pi user)
	
	AllowGroups pi mysqlusers
	Match Group mysqlusers
	        PermitOpen 127.0.0.1:3306
	        X11Forwarding no
	        PermitTTY no
	        ForceCommand /bin/false

g. $ sudo service ssh restart (this will restart ssh server with new rules)
h. $ sudo mysql
k. CREATE USER 'mysqltester'@'localhost' IDENTIFIED BY 'password'; (create new user which will have restricted permissions.)
l. GRANT ALL PRIVILEGES ON testdb.* TO 'mysqltester'@'localhost';
m. FLUSH PRIVILEGES;
n. can now give testuser + pwd and mysqltest + pwd to someone and they can securely connect to the specified database with all privages but have no other access to the server
    	$ ssh testuser@serverpi -L 3307:127.0.0.1:3306 -N
    	$ mysql --host=127.0.0.1 --port=3307 -u mysqltester -p
    	mysql> use testdb;

### next steps
a. replace ssh user names and passwords with public and private keys
