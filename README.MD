Instructions to run this project:

Step 1: Start A New Ubuntu Linux Server Instance On Amazon Lightsail:

	(i). First, log in to Lightsail. If you don't already have an Amazon Web Services account, you'll be prompted to create one.

	(ii). Once you're logged in, Lightsail will give you a friendly message with a robot on it, prompting you to create an instance. A 		      Lightsail instance is a Linux server running on a virtual machine inside an Amazon datacenter.

	(iii). Choose Linux/Unix platform, OS Only and Ubuntu 16.04 LTS.

	(iv). Choose an instance plan.

	(v). Give the instance a hostname.

Step 2: SSH into the server:

	(i). Download Default Private Key from the Account menu on Amazon Lightsail.

	(ii). Move the private key file into the folder ~/.ssh . So if you downloaded the file to the Downloads folder, just execute the 	       following command in your terminal. mv ~/Downloads/LightsailDefaultKey-ap-south-1.pem ~/.ssh/.

	(iii). Open your terminal and type in chmod 600 ~/.ssh/LightsailDefaultKey-ap-south-1.pem

	(iv). In your terminal, type:
		ssh -i ~/.ssh/LightsailDefaultKey-ap-south-1.pem ubuntu@52.66.195.139

Step 3: Secure your server:
	(i). Update all currently installed packages.

	(ii).Change the SSH port from 22 to 2200:

		(a) By typing sudo vim /etc/ssh/sshd_config in the terminal change the port 22 to 2200.
		(b) Reload SSH by typing sudo service ssh restart

	(iii). Configure the Uncomplicated Firewall (UFW) only allow incoming connections for 
		SSH (port 2200), HTTP (port 80), and NTP (port 123) by typing:

		(a) sudo ufw allow 2200/tcp
		(b) sudo ufw allow 80/tcp
		(c) sudo ufw allow 123/tcp
		(d) sudo ufw enable 
	
	(iv). In your terminal, type:
		ssh -i ~/.ssh/LightsailDefaultKey-ap-south-1.pem ubuntu@52.66.195.139 -p 2200

Step 4: Give grader access:

	(i). Add the user grader by typing sudo adduser grader

	(ii). Give grader the permission to sudo :

		(a) sudo touch /etc/sudoers.d/grader
		(b) sudo vim /etc/sudoers.d/grader and add this 'grader ALL=(ALL:ALL) ALL' in the file.

	(iii). Create an SSH key pair for grader using the ssh-keygen tool:

		(a) On local machine run ssh-keygen
		(b) Enter file in which to save the key(in my case /home/mustaque/.ssh/grader_key)
		(c) Enter passphrase twice.
		(d) Run cat ~/.ssh/grader_key.pub and copy the contents of the file
		(e) Log In as the user grader in the virtual machine.
		(f) Create a directory by running mkdir .ssh
		(g) Create a file touch .ssh/authorized_keys.
		(h) Run vim .ssh/authorized_keys and paste the content generated in the local machine(from the file ~/.ssh/grader_key.pub ) in 			this file.
		(i) Run chmod 700 .ssh and chmod 644 .ssh/authorized_keys
		(j) Run sudo service ssh restart
		(k) On the local machine run:  
			ssh -i ~/.ssh/grader_key grader@52.66.195.139 -p 2200

Step 5: Prepare to deploy the project:

	(i). Configure the local timezone to UTC by running sudo dpkg-reconfigure tzdata

	(ii). Install and configure Apache to serve a Python mod_wsgi application:

		(a) Install Apache by running sudo apt-get install apache2
		(b) Install mod_wsgi by running sudo apt-get install python-setuptools libapache2-mod-wsgi
		(c) Restart Apache by running sudo service apache2 restart

	(iii). Install and configure PostgreSQL:

		(a) Install PostgreSQL by running sudo apt-get install postgresql
		(b) Make sure no remote connections are allowed by checking the file sudo vim /etc/postgresql/9.3/main/pg_hba.conf
		(c) Login as user "postgres" by running sudo su - postgres
		(d) Run psql
		(e) Create a new database named catalog and create a new user named catalog in postgreSQL shell by running:
			postgres=# CREATE DATABASE catalog;
			postgres=# CREATE USER catalog;
		(f) Set a password for user catalog by running:
			postgres=# ALTER ROLE catalog WITH PASSWORD 'catalog';
		(g) Give user "catalog" permission to "catalog" application database by running:
			postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
		(h) Quit postgres and exit from user 'postgres'.

	(iv). Install git by running sudo apt-get install git (if git is already installed, ignore this step)

Step 6: Deploy the Item Catalog project:

	(i). Clone and setup your Item Catalog project from the Github repository:

		(a) Move to the /var/www directory by running cd /var/www
		(b) Create the application directory by running sudo mkdir FlaskApp
		(c) Move to FlaskApp directory using cd FlaskApp
		(d) Clone the Catalog App to the virtual machine by running sudo git clone https://github.com/mhghy93/catalog-application.git
		(e) Rename the project's name by running sudo mv ./catalog-application ./FlaskApp
		(f) Move to the inner FlaskApp directory using cd FlaskApp
		(g) Rename application.py to __init__.py using sudo mv application.py __init__.py
		(h) In __init__.py and database_setup.py :
			replace engine = create_engine('sqlite:///catalogsitem.db',connect_args={'check_same_thread': False}) 	
			with engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog')
		(i) Also in __init__.py:
			replace app.debug = True and app.run(host='0.0.0.0', port=8000)
			with app.run()
		(j) Install pip by running sudo apt-get install python-pip
		(k) Use pip to install dependencies
			- sudo pip install sqlalchemy flask-sqlalchemy psycopg2 bleach requests
			- sudo pip install flask packaging oauth2client redis passlib flask-httpauth
		(l) Install psycopg2 by running sudo apt-get -qqy install postgresql python-psycopg2
		(m) Create database schema by running sudo python database_setup.py

	(ii) Configure and Enable a New Virtual Host:
		
		(a) Create FlaskApp.conf by running sudo touch /etc/apache2/sites-available/FlaskApp.conf
		(b) Add the following lines in FlaskApp.conf:

			<VirtualHost *:80>
				ServerName 52.66.195.139
				ServerAdmin hussainmustaqueghy93@gmail.com
				WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
				<Directory /var/www/FlaskApp/FlaskApp/>
					Order allow,deny
					Allow from all
				</Directory>
				Alias /static /var/www/FlaskApp/FlaskApp/static
				<Directory /var/www/FlaskApp/FlaskApp/static/>
					Order allow,deny
					Allow from all
				</Directory>
				ErrorLog ${APACHE_LOG_DIR}/error.log
				LogLevel warn
				CustomLog ${APACHE_LOG_DIR}/access.log combined
			</VirtualHost>
		(c) Enable the virtual host with the following command: sudo a2ensite FlaskApp

	(iii) Create the .wsgi File:
		
		(a) Create the .wsgi File under /var/www/FlaskApp:
		
			cd /var/www/FlaskApp
			sudo touch flaskapp.wsgi 

		(b) Add the following lines of code to the flaskapp.wsgi file:

			#!/usr/bin/python
			import sys
			import logging
			logging.basicConfig(stream=sys.stderr)
			sys.path.insert(0,"/var/www/FlaskApp/")

			from FlaskApp import app as application
			application.secret_key = 'Add secret key'

	(iv) Restart Apache by running sudo service apache2 restart

Step 7: Visit http://52.66.195.139/






		




		
