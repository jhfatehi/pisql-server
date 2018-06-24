# pisql-server
## setup a mysql server on a raspberry pi with some security

### introduction
The purpose of this exercise is to setup a MySQL server on a RPI and connecting to it through a SSH tunnel on your local network.  The server and client are both Linux distributions but I do have a few notes about using a Windows client.  I used Ubuntu 16.04 as my client.  I am not a Linux wiz so I'm not sure if there would be any difference with the default SSH configuration using a different server distribution.  At the end there is a bit on adding more security which is not really required on your local network but good to learn for servers on the internet like DigitalOcean.

### make a bootable rasbian sd
1. I did this exercise on a 2 GB SD-card and it worked but it's close.  Get 4 GB or greater if you are buying a new card.
1. Download [Rasbian](https://www.raspberrypi.org/downloads/raspbian/).  This is a RPI specific Linux distribution based on Debian.  The lite version does not include a desktop environment (GUI) which is fine for this exercise.  If you want the desktop environment get the version with desktop but make sure your SD-card is at least 8 GB.
1. I like [Etcher](https://etcher.io/) for burning bootable media.

### configue remote access
1. Boot up the RPI with SD-card, monitor, and keyboard.
1. Login with default credentials which are user name **pi** and password **raspberry**.  Note that the root login is not permitted by default.
1. Go to the RPI configuration menu.  Use **Enter** for select and **Escape** for back.  If you are not using Rasbian you will have to research how to complete the following steps separately.

   `$ sudo raspi-config` 
   
1. Change password of **pi** user.
1. Change the hostname.  I use the hostname **serverpi**.

   `network options -> hostname`
   
1. Enable Wi-Fi.  This is not required if a wired connection is being use, which is a good idea if possible.  If you are using a RPI v1 or v2 you will need a Wi-Fi dongle.

   `network options -> wi-fi`

1. Enable the SSH server.

   `interfacing options -> ssh -> enable`
   
1. **Esc** to exit config menu.
1. Get the IP address of the RPI.  The IP will not be needed because the RPI can be addressed by the hostname but since we are here it's a good idea to grab it now.

   `$ ip addr show`
   
1. The monitor and keyboard are no longer required.

### login though ssh
1. From your client computer (any other computer on your network) login to the RPI as the **pi** user via SSH.  There will be a warning about this being the first time connecting to a new computer which is the RPI in this case so say yes.  Most Linux distributions comes with OpenSSH client.  If yours does not you will need to research how to install it.

   `$ ssh pi@serverpi`
   
* *Windows* - You can use PuTTY to login via SSH.  Using the IP address will work in PuTTY.  The RPI should keep same IP if router is not re-started.  It might keep same address after a restart too.  I'm guessing the hostname instead of IP will work in PuTTY but I have not tried it.

### install and configure mysql 
1. SSH into **serverpi** as the **pi** user.

   `$ ssh pi@serverpi`
   
1. Install MySQL server.  In Raspbian this will actually install MariaDB which is [practical the same](https://blog.panoply.io/a-comparative-vmariadb-vs-mysql) as MySQL.  The default root password for MySQL is blank.  When running the *mysql_secure_installation* script do not change the root password and say yes to all other options.

   `$ sudo apt-get install mysql-server`  
   `$ sudo mysql_secure_installation`
  
1. Open MySQL as the root user and create a new user with read write access to all databases.  Substitute **mysqluser1** and **password** with a pair of your choosing or use as is.  Then commit the privilege change and exit the MySQL shell.

   `$ sudo mysql`   
   `mysql> CREATE USER 'mysqluser1'@'localhost' IDENTIFIED BY 'password';`  
   `mysql> GRANT ALL PRIVILEGES ON *.* TO 'mysqluser1'@'localhost';`  
   `mysql> FLUSH PRIVILEGES;`  
   `mysql> EXIT;`
   
1. You can now open the MySQL shell as a non-root user.

   `$ mysql -u mysqluser1 -p`

### use mysql through ssh tunnel from client
1.  Open a SSH tunnel from the client.  The phrase after *-L* is in the format *local_socket:host:hostport*.  This binds localhost (127.0.0.1) port 3306 (the default MySQL port) from **serverpi** to the client's port 3307.  The *-N* means *Do not execute a remote command.  This is useful for just forwarding ports.*  If the terminal window is closed the tunnel will also close.  I am using port 3307 on the client instead of 3306 in case there is a local MySQL server running on the client which is already using port 3306.  Use the **pi** user password when prompted.

    `$ ssh pi@serverpi -L 3307:127.0.0.1:3306 -N`
   
1. If you do not have a MySQL client installed do it now.  If you already have a MySQL client you can skip this step.

   `$ sudo apt install mariadb-client`

1. You can now access the MySQL server on the RPI from your client.  Use the **mysqluser1** user password when prompted.

   `$ mysql --host=127.0.0.1 --port=3307 -u mysqluser1 -p `  
   `mysql> EXIT;`
   
1. A database can be created by running an [SQL script](https://github.com/jhfatehi/pisql-server/blob/master/testdb_schema.sql).  Run the command from the dirrectory where you saved the script or specify the full script path.  In the provided script a database called **testdb** will be created.  If a database named **testdb** already exists the script will drop it.

   `$ mysql --host=127.0.0.1 --port=3307 -u mysqluser1 -p < testdb_schema.sql`
   
* For future reference, it is also possible to run a script against and existing database.

   `$ mysql --host=127.0.0.1 --port=3307 -u mysqluser1 -p db_name < script_to_run.sql`
   
* *Windows* - PuTTY [instructions](https://www.linode.com/docs/databases/mysql/create-an-ssh-tunnel-for-mysql-remote-access/) for the tunnel.

### add some more security
1. SSH into **serverpi** as the **pi** user.

   `$ ssh pi@serverpi`

1. Create a group for MySQL users, create a user in that group, and assign a password to that user.  Substitute **mysqlgroup** and **piuser2** with names of your choosing or use as is.

   `$ sudo groupadd mysqlgroup`  
   `$ sudo useradd -g mysqlgroup piuser2`  
   `$ sudo passwd mysqluser2`

* For future reference this will let you delete a user along with the user's home directory and mail spool.

   `$ sudo userdel -r username`

1. Open the SSH config file, add the following block of text to the bottom, and restart the SSH server to apply the changes.  This will control access by groups and then restrict assess of **mysqlgroup**.  The last line removes shell access over SSH from **mysqlgroup**.  Adding the **pi** group preserves SSH access for the **pi** user.  The *PermitOpen* limits the port that **mysqlgroup** can bind to.  With out this line **mysqlgroup** could bind to any port.

   `$ sudo nano /etc/ssh/sshd_config`
	
   >AllowGroups pi mysqlgroup  
   >Match Group mysqlgroup  
   >&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;PermitOpen 127.0.0.1:3306  
   >&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;X11Forwarding no  
   >&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;PermitTTY no  
   >&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ForceCommand /bin/false
   
   `$ sudo service ssh restart`

1. Open MySQL shell as root and create a new user that has read write access only to the **testdb** database.

   `$ sudo mysql`  
   `mysql> CREATE USER 'mysqluser2'@'localhost' IDENTIFIED BY 'password';`  
   `mysql> GRANT ALL PRIVILEGES ON testdb.* TO 'mysqluser2'@'localhost';`  
   `mysql> FLUSH PRIVILEGES;`  
   `mysql> EXIT;`
   
1. The user **piuser2** can now use the **mysqluser2** user to access **testdb** but no other databases.  The user **piuser2** has no other access in the RPI.  At least that's the goal.

   `$ ssh piuser2@serverpi -L 3307:127.0.0.1:3306 -N`  
   `$ mysql --host=127.0.0.1 --port=3307 -u mysqluser2 -p`  
   `mysql> USE testdb;`

### next steps
1. Replace user names and passwords with public and private keys for SSH access.
1. Create a MySQL user that has full administrative privileges so root login to MySQL is not required.
1. Repeat this exercise with Ubuntu Server or CentOS.
