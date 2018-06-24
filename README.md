# pisql-server
## setup a mysql server on a raspberry pi with some security

### introduction
The purpose of this exercise is to setup a MySQL server on a RPI and connecting to it through a SSH tunnel on your local network.  The server and client are both linux distributions but I do have a few notes about using a Windows client.  I used Ubuntu 16.04 as my client.  I am not a linux wiz so I'm not sure if there would be any difference with the default SSH configuration using a different server distribution.  At the end there is a bit on adding more security which is not really required on your local network but good to learn for servers on the internet like DigitalOcean.

### make a bootable rasbian sd
1. I did this exercise on a 2 GB sd-card and it worked but it's close.  Get 4 GB or greater if you are buying a new card.
1. Download [Rasbian](https://www.raspberrypi.org/downloads/raspbian/).  This is a RPI specific lnius distribution based on Debian.  The lite version does not include a desktop environment (GUI) which is fine for this exercise.  If you want the dsktop environment get the version with desktop but make sure your sd-card is at least 8 GB.
1. I like [Etcher](https://etcher.io/) for buring bootable media.

### configue remote access
1. Boot up the RPI with sd-card, monitor, and keyboard.
1. Login with default credentials - pi/raspberry.  Note that the root login is not permitted by default.
1. Go to the RPI configuration menu.  Use Enter for select and Escape for back.  If you are not using rasbian you will have to research how to complete the following steps seperatly.

   `$ sudo raspi-config` 
   
1. Change password of 'pi' user.
1. Change the hostname.  I use the hostname 'serverpi'.

   `network options -> hostname`
   
1. Enable WiFi.  This is not required if a wired connection is being use, which is a good idea if possible.  If you are using a RPI v1 or v2 you will need a WiFi dongle.

   `network options -> wi-fi`

1. Enable the SSH server.

   `interfacing options -> ssh -> enable`
   
1. Esc to exit config menu
1. Get the IP address of the RPI.  The IP will not be needed because of hostname but since we are here it's a good idea to grab IP now.

	`$ ip addr show`

### login though ssh
1. From your client computer (any other computer on your network) login to the RPI at the 'pi' user via SSH.  There will be a warning about this being the first time connecting to a new computer which is the RPI in this case so say yes.  Most linux distributions comes with OpenSSH client.  If yours does not you will need to research how to install it.

   `$ ssh pi@serverpi`
   
* Windows - You can use PuTTY to login via SSH.  Using the IP address will work in PuTTY.  The RPI should keep same IP if router is not re-started.  It might keep same address after a restart too.  I'm guessing the hostname instead of IP will work in PuTTY but I have not tried it.

### install and configure mysql 
1. SSH into 'serverpi' as the 'pi' user.
1. Install MySQL server.  In Raspbian this will actuall install MariaDB which is [practicaly the same](https://blog.panoply.io/a-comparative-vmariadb-vs-mysql) as MySQL.  The default root password is blank.  When running the 'mysql_secure_installation' do not change the root password and say yes to all other options.

   `$ sudo apt-get install mysql-server`  
   `$ sudo mysql_secure_installation`
  
1. Open MySQL as the root user and create a new user with read write access to all databases.  Substiute the username and password.  I use the username bob.  Then commit the privilege change and exit the MySQL shell.

   `$ sudo mysql (open mysql shell as root user.)`   
   `mysql> GRANT ALL PRIVILEGES ON *.* TO 'username'@'localhost' IDENTIFIED BY 'password';`   
   `mysql> FLUSH PRIVILEGES;`  
   `exit;`
   
1. You can now open the MySQL shell as a non-root user.

   `$ mysql -u bob -p`

### use mysql through ssh tunnel from client
1.  The phrase after '-L' is in the format 'local_socket:host:hostport'.  This binds localhost (127.0.0.1) port 3306 (the default MySQL port) from serverpi to the client's port 3307.  The '-N' means 'Do not execute a remote command.  This is useful for just forwarding ports.'  This will open the tunnel.  If the terminal window is closed the tunnel will also close.  I am using port 3307 on the client instead of 3306 incase there is a local mysql server running on client which is using port 3306.  Use the 'pi' user password when prompted.

   `$ ssh pi@serverpi -L 3307:127.0.0.1:3306 -N`
   
1. If you do not have a MySQL clinet installed do it now.

   `$ sudo apt install mariadb-client`

1. You can now access the MySQL server on the RPI from your client.  Use the 'bob' user password when prompted.

   `$ mysql --host=127.0.0.1 --port=3307 -u bob -p `  
   `exit;`
   
1. A database can be ceated by running an [SQL script](https://github.com/jhfatehi/pisql-server/blob/master/testdb_schema.sql).

   `$ mysql --host=127.0.0.1 --port=3307 -u bob -p < testdb_schema.sql`
   
* For future reference, it is also possible to run a script against and existing database.

   `$ mysql --host=127.0.0.1 --port=3307 -u bob -p db_name < script_to_run.sql`
   
* Windows - PuTTY [instructions](https://www.linode.com/docs/databases/mysql/create-an-ssh-tunnel-for-mysql-remote-access/) for the tunnel.

### add some more security
1. SSH into 'serverpi' as the 'pi' user.
1. Create a group for MySQL users, create a user in that group, and assign a password to that user.  Substiute the groupname and username.  I user the groupname 'mysqlusers' and I use the username 'bob'.

   `$ sudo groupadd groupname`  
   `$ sudo useradd -g groupname username`  
   `$ sudo passwd username`

* For future reference this will let you delete a user + home dirrectory and mail spool.

   `$ sudo userdel -r username`

1. Open the SSH config file and add the following block fo text to the bottom.  This will control access by groups and then resrict assess of mysqlusers group.  The last line removes shell access over SSH from 'mysqlusers'.  Adding the 'pi' group preserves SSH access for the 'pi' user.  The 'PermitOpen' limits the port that 'mysqlusers' can bind to.  With out this line 'mysqlusers' could bind to any port.

   `$ sudo nano /etc/ssh/sshd_config`
	
   >AllowGroups pi mysqlusers  
      >Match Group mysqlusers  
	        PermitOpen 127.0.0.1:3306  
	        X11Forwarding no  
	        PermitTTY no  
	        ForceCommand /bin/false

1. $ sudo service ssh restart (this will restart ssh server with new rules)
1. $ sudo mysql
1. CREATE USER 'mysqltester'@'localhost' IDENTIFIED BY 'password'; (create new user which will have restricted permissions.)
1. GRANT ALL PRIVILEGES ON testdb.* TO 'mysqltester'@'localhost';
1. FLUSH PRIVILEGES;
1. can now give testuser + pwd and mysqltest + pwd to someone and they can securely connect to the specified database with all privages but have no other access to the server
    	$ ssh testuser@serverpi -L 3307:127.0.0.1:3306 -N
    	$ mysql --host=127.0.0.1 --port=3307 -u mysqltester -p
    	mysql> use testdb;

### next steps
1. Replace user names and passwords with public and private keys for SSH access.
1. Create a MySQL user that has full administrative privileges so root login to MySQL is not required.
