# pisql-server
##Setup a MySQL server on a RPI with some security.
###1. make a bootable rasbian sd
	a. 2gb sd card works but it's close.  get 4gb or greater if buying a new card.
	b. https://etcher.io/ (to burn sd)
	c. https://www.raspberrypi.org/downloads/raspbian/ (choose lite)

###2. configue remote access
	a. boot up pi with monitor and keyboard
	b. login with default credentials - pi/raspberry (root login is not permitted by default)
	c. $ sudo raspi-config (enter for select and escape for back)
	d. change password of 'pi' user
	e. network options -> hostname (i use the hostname serverpi)
	f. network options -> wi-fi (not required if wired connection is being use, which is a good idea if possible)
	g. interfacing options -> ssh -> enable
	h. esc to exit config menu
	i. $ ip addr show (ip will not be needed because of hostname but good idea to grab ip now)

###3. login though ssh
	a. *Ubuntu* $ ssh pi@serverpi (there will be a warning about this being the first time connecting to computer so make sure you know what it is.)
	b. *Windows* need to have putty.  ip address will work in putty.  pi should keep same ip if router is not re-started.  might keep same address after restart too.  hostname might work in putty but i have not tried

###4. install and configure mysql 
	a. ssh into serverpi
	b. $ sudo apt-get install mysql-server
	c. $ sudo mysql_secure_installation (default root password is blank.  no not chnage root password and say yes to all other options)
	d. $ sudo mysql (open mysql shell as root user.  shell will say MariaDB which is almost the same as mysql)
	e. GRANT ALL PRIVILEGES ON *.* TO 'username'@'localhost' IDENTIFIED BY 'password'; (create a new user will full privliges)
	f. exit; (exits mysql)
	g. $ mysql -u username -p (mysql shell can now be opened without root)

###5. use mysql through ssh tunnel from Ubuntu client
	a. $ ssh pi@serverpi -L 3307:127.0.0.1:3306 -N (use pi user password.  this will open the tunnel.  if the bash shell is closed the tunnel will also close.  usering port 3307 instead of default 3306 incase ther is a local mysql server running on client)
	b. $ mysql --host=127.0.0.1 --port=3307 -u username -p (will open mysql shell.  use mysql username password.  this will require a mysql client be installed [$ sudo apt-get install mysql-client])
	c. $ mysql --host=127.0.0.1 --port=3307 -u username -p < testdb_schema.sql (script needs to start with 'USE db_name')
	d. $ mysql --host=127.0.0.1 --port=3307 -u username -p db_name < script_to_run.sql (script will run on db_name)
	e. *Windows* here are putty instructions for the tunnel https://www.linode.com/docs/databases/mysql/create-an-ssh-tunnel-for-mysql-remote-access/

###6. add some security - not really required on local network but good to learn servers on the internet, like digitalocean
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

 ###7. next steps
 	a. replace ssh user names and passwords with public and private keys
