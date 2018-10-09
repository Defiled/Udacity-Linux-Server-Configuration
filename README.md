# Udacity-Linux-Server-Configuration
Udacity Linux Server Configuration

1. Update all packages
- `sudo apt-get update`
- `sudo apt-get dist-upgrade`


2. Create a user called "grader"
- `sudo adduser grader`


3. Give grader admin rights
- `sudo usermod -aG sudo grader`
- `su - grader’ && ‘sudo whoami` // validate output is 'root'

4. Setup SSH key for grader
a. Create the directory and set file permissions
- `mkdir -p $HOME/.ssh` // Create a .ssh directory
- `chmod 0700 $HOME/.ssh` // Modify permissions so only grader can read, write, and execute
b. Temporarily allow password authentication to easily copy over ssh key
- `sudo nano /etc/ssh/sshd_config`
- Set PasswordAuthentication to yes (if not yes by default), press 'CTRL + X' and 'y' to exit and confirm save
- `sudo service ssh restart` // Restart ssh service for changes to take effect
- Switch to local machine you intend to ssh into server from...
- `ssh-keygen -t rsa` // Generates an ssh key of type RSA in a directory of your choice e.g /Users/waldo/.ssh/
- `ssh-copy-id -i $HOME/.ssh/grader.pub grader@52.87.206.125` // Replace with location of your rsa key and your server's IP
- `sudo nano /etc/ssh/sshd_config`
- Set PasswordAuthentication back to no to force SSH then press 'CTRL + X' and 'y' to exit and confirm save
- `sudo service ssh restart`
- `ssh grader@52.87.206.125 -i ~/.ssh/grader` // SSH into server to confirm key is working
c. Change default SSH port to 2200
- `sudo nano /etc/ssh/sshd_config` 
- Replace ‘#Port 22’ with ‘Port 2200’ 
- `sudo systemctl restart ssh`
d. Edit your firewall rules in the LightSail networking tab
- Add a new rule, select 'Custom' for the Application column and '2200' for the port range
- Delete the first rule which allows SSH on Port 22
e. Check SSH is working on Port 2200
- Back in the terminal, type `systemctl ssh restart && exit`
- `ssh grader@52.87.206.125 -p 2200 -i ~/.ssh/grader`


5. Configure Firewall Settings
- `sudo ufw default deny incoming; sudo ufw default allow outgoing;`
- `sudo ufw allow 2222/tcp; sudo ufw allow http; sudo ufw allow ntp`
- `sudo ufw enable && sudo ufw status` // Turn on firewall and validate ports are set up correctly
- `exit` and then `ssh grader@54.152.4.245 -p 2200 -i ~/.ssh/grader` to ensure its working


6. Set timezone to UTC
- `timedatectl set-timezone UTC`
- To validate: `timedatectl status`


7. Install Apache and mod_wsgi 
- `sudo apt-get install apache2`
- `curl http://localhost` to validate apache is online
- `sudo apt-get install libapache2-mod-wsgi python-dev`
- Restart apache `systemctl restart apache2`
- source: https://www.digitalocean.com/community/tutorials/how-to-set-up-an-apache-mysql-and-python-lamp-server-without-frameworks-on-ubuntu-14-04


8. Install and setup PostgreSQL database with catalog user
- `sudo apt-get install postgresql postgresql-contrib`
- `sudo apt-get install libpq-dev python-dev`
- `sudo nano /etc/postgresql/10/main/pg_hba.conf` // Make sure remote connections are disabled (should be by default)
- `sudo su postgres`
- `ALTER USER postgres WITH PASSWORD 'postgres’;’
- `CREATE USER catalog WITH PASSWORD 'catalog’;`
- `ALTER USER catalog CREATEDB SUPERUSER;`
- `CREATE DATABASE pokedex WITH OWNER catalog;`
- `\c pokedex`
- `REVOKE ALL ON SCHEMA public FROM public;`
- `GRANT ALL ON SCHEMA public TO catalog;`
- Check everything with `\du` and `\l`
- Quit with `\q` and return to grader user with `exit`


9. Setup Python project
a. Install Git
- `sudo apt install git-all`
b. Setup directory
- `cd /var/www/html` // Directory you want to clone repository into
- `Git clone https://github.com/Defiled/Pokedex.git`
- `sudo chown -R grader:grader /var/www/Pokedex/`
c. Install libraries, tools and dependencies 
- `sudo apt-get -qq install python python-pip`
- `sudo pip install flask sqlalchemy flask-sqlalchemy psycopg2-binary httlib2 oauth2client requests`
d. Convert to PostgreSQL from SQLite
- `sudo nano /var/www/Pokedex/db_populate.py`
- Change "engine = create_engine('sqlite:///pokedex.db’)” to “engine = create_engine('postgresql://catalog:catalog@localhost/pokedex’)”
- Do the same in db_populate.py and project.py


10. Configure Apache and mod_wsgi
a. Setup Apache config file
- `sudo nano /etc/apache2/sites-available/Pokedex.conf`
- Insert the following:
```
<VirtualHost *:80>
   ServerName 52.20.134.48
   # ServerAlias pokedex.usa
   ServerAdmin grader@52.20.134.48
   WSGIDaemonProcess catalog python-path=/var/www/Pokedex:/usr/local/lib/python2.7/dist-packages
   WSGIProcessGroup catalog
   WSGIScriptAlias / /var/www/Pokedex/pokedex.wsgi
   <Directory /var/www/>
       Order allow,deny
       Allow from all
   </Directory>
   Alias /static /var/www/Pokedex/static
   <Directory /var/www/Pokedex/static/>
       Order allow,deny
       Allow from all
   </Directory>
   ErrorLog ${APACHE_LOG_DIR}/error.log
   LogLevel warn
   CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- `systemctl reload apache2` // Restart the apache server
- `sudo a2ensite Pokedex` // Enable site
b. Create .wsgi script file
- `sudo nano pokedex.wsgi` 
- Insert the following:
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/Pokedex/")
sys.path.insert(1, "/var/www/")

from Pokedex import app as application
application.secret_key = 'pikachu'
```
c. Project tweaks
- `sudo mv project.py __init__.py` so that python knows to treat the Pokedex directory as a module
- `sudo nano __init__.py` and update the CLIENT_ID variable to load the absolute path of the file it now lives in


11. Finish setup
a. Setup and populate the PostgreSQL database
- `Python db_setup.py`
- `Python db_populate.py`
- `systemctl reload apache2`
b. Connect to server
- `sudo cat /var/log/apache2/error.log` to debug any issues


Extra
- Update DNS and have my purchased domain point to public IP address
- Add domain to facebook and oauth developer console and update secrets
http://xip.io/
