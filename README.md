# Linux server Configuration
Deploying a Flask application on Ubuntu. 
This file contains the software and steps required to deploy a Flask app.
### Introduction
For this application Digital Ocean was used for a Ubuntu Linux server instance. 
- IP Address: 167.71.128.240
- URL: http://www.167.71.128.240.xip.io/
- App directory (WSGIScript File in/web root path): /var/www/catalog/
- SSH port: 2200
- ssh grader@167.71.128.240 -p 2200
- 'grader' user passphrase: thisisroot
- 'grader' user password: root
### Initial steps
logging into root@167.71.128.240
- create local key using "ssh-keygen"
- copy the id_rsa.pub file content to digitalocean for access to root
- run command "ssh root@167.71.128.240"
### Create new user
- $ sudo adduser grader
- give sudo permissions: $ usermod -aG sudo grader
- log into grader: $ su - grader 
### Update packages
- $ sudo apt-get update
- $ sudo apt-get upgrade
### Install Apache
- Run command: $ sudo apt-get install apache2
- Go to 167.71.128.240 to see if Apache has installed and running
### Install mod_wsgi
- $ sudo apt-get install libapache2-mod-wsgi python-dev
- $ sudo a2enmod wsgi [this will enable wsgi in case it did not from the previous command]
- $ sudo service apache2 start
### Install Git and clone the required repo
- $ sudo apt-get install git
- $ cd /var/www 
- $ sudo mkdir catalog 
- $ sudo chown grader:grader catalog
- $ cd catalog 
- $ git clone https://github.com/thelonehegelian/items-catalog-udacity catalog
##### Make .git directory unaccessible: 
- cd /var/www/catalog/catalog/.git
- $ sudo nano .htaccess: Add "RedirectMatch 404 /\.git" to the .htaccess file
### Install Virtual Environment and dependencies
- cd /var/www/catalog
- $ sudo apt-get install python-pip 
Install dependencies outside the virtual environment
- $ sudo pip install -r catalog/requirements.txt
Install the virtual environment
- $ sudo pip install virtualenv 
- $ sudo virtualenv venv 
- $ source venv/bin/activate 
- $ sudo chmod 777 venv
### Configure New Virtual Host
- $ sudo nano /etc/apache2/sites-available/catalog.conf
- Add the lines:

<VirtualHost *:80>

        ServerName 167.71.128.240
        ServerAlias 167.71.128.240
        ServerAdmin grader@167.71.128.240
       WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
       WSGIProcessGroup catalog
       WSGIScriptAlias / /var/www/catalog/catalog.wsgi
       <Directory /var/www/catalog/catalog/>
           Order allow,deny
           Allow from all
       </Directory>
       Alias /static /var/www/catalog/catalog/static
       <Directory /var/www/catalog/catalog/static/>
           Order allow,deny
           Allow from all
       </Directory>
       ErrorLog ${APACHE_LOG_DIR}/error.log
       LogLevel warn
       CustomLog ${APACHE_LOG_DIR}/access.log combined
    
    
    </VirtualHost>
    
- Save and Exit
### Enable the new virtual host
-   $ sudo a2ensite catalog
-   $ systemctl reload apache2
### Configure the .wsgi File
- $ cd /var/www/catalog/
- $ sudo nano catalog.wsgi
- Add the following:
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog/")

    from catalog import app as application
    application.secret_key = 'supersecretkey'

### Edit item catalog program files
Rename server.py to __init__.py
- cd /var/www/catalog/catalog
- $ mv server.py __init__.py
- Update the absolute path of client_secrets.json in __init__.py to '/var/www/catalog/catalog/client_secrets.json'
### Install and configure PostgreSQL
- $ sudo apt-get install libpq-dev python-dev
- $ sudo apt-get install postgresql postgresql-contrib
- $ sudo -u postgres psql
- CREATE USER catalog with password 'catalog';
- ALTER USER catalog CREATEDB;
- CREATE DATABASE catalog with owner catalog;
- Connect to database: \c catalog
- REVOKE ALL ON SCHEMA public FROM public;
- GRANT ALL ON SCHEMA public to catalog;
- Log out from postgres: \q
### Edit python files
- Change in database_setup.py, books.py and __init__.py : engine = create_engine('sqlite:///bookstore.db') to engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
### Populate the database and run the website
- $ python books.py
- Wait for the message "Books Added"
- Restart Apache to launch the app: $ sudo service apache2 restart
- Go to 167.71.128.240. The app should be working now.
- For debugging: $ sudo tail -20 /var/log/apache2/error.log
#### Last checks
- these two might not have installed through the requirement.txt, Install now if there are errors:	
- $ pip install SQLAlchemy
- $ pip install psycopg2
- check if remote connections to Postgresql are disabled: $ sudo nano /etc/postgresql/10/main/pg_hba.conf
Permissions should look like this: 

- | local         | replication   | |      all      |     peer      | 
- | host          | replication   | | 127.0.0.1/32  |      md5      |
- | host          | replication   | |  ::1/128      |      md5      |

- Source: https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
### Enable ssh connection
Shut the virtual environment
- $ deactivate
Load the public key on the virtual machine
- $ sudo mkdir .ssh
- $ sudo touch .ssh/authorized_keys
- $ sudo nano .ssh/authorized_keys
- Copy and the public key on local machine
### Change Permsissions
/home/grader/.ssh and /home/grader/.ssh/authorized_keys
$ sudo chown -R  grader:grader /home/grader/.ssh
$ sudo chmod 700 /home/grader/.ssh
$ sudo chmod 600 /home/grader/.ssh/authorized_keys
### Enable Key authentication and disable password
- run: $ sudo nano /etc/ssh/sshd_config
- Change lines: 
PubkeyAuthentication yes
remove # from "AuthorizedKeysFile      .ssh/authorized_keys .ssh/authorized_keys2" line
PasswordAuthentication no
- Save and exit
- $ service ssh restart
- $ exit
- ssh to grader to confirm the key-based authentication works 
### Disable root login
- $ sudo nano /etc/ssh/sshd_config
- change: PermitRootLogin no
### Change default port 
- $ sudo nano /etc/ssh/sshd_config
- change line #Port 22 to Port 2200 (remove the #)
- save and exit
- $ service ssh restart
### Set up Firewall
- $ sudo ufw allow 2200/tcp
- $ sudo ufw allow 80/tcp
- $ sudo ufw allow 123/udp
- $ sudo ufw enable
- $ sudo ufw status [to doublecheck]
- $ service ssh restart
### Configure Timezone
- $ sudo dpkg-reconfigure tzdata
### Update packages
- $ sudo apt-get update
- $ sudo apt-get upgrade
### Configuring system for automatic updates
Install the 'unattended-upgrades' package
- $ sudo apt install unattended-upgrades
- $ sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
Uncomment line:
- "${distro_id}:${distro_codename}-updates";
Uncomment line and change 'false' to 'true'
- Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
- Unattended-Upgrade::Automatic-Reboot "true";
Uncomment and change Reboot time
- Unattended-Upgrade::Automatic-Reboot-Time "02:38";
#### Enable automatic update and set intervals
- $ sudo nano /etc/apt/apt.conf.d/20auto-upgrades
Copy and paste the following lines: 
- APT::Periodic::Update-Package-Lists "1";
- APT::Periodic::Download-Upgradeable-Packages "1";
- APT::Periodic::AutocleanInterval "7";
- APT::Periodic::Unattended-Upgrade "1";
Debug to ensre if it worked: 
- $ sudo unattended-upgrades --dry-run --debug
Log can also be checked in a few days to see if the upgrades worked: 
- $ sudo cat /var/log/unattended-upgrades/unattended-upgrades.log
- Source: https://libre-software.net/ubuntu-automatic-updates/
### Sources
- digitalocean.com
- https://stackoverflow.com
- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
- https://linux.die.net
- https://www.ostechnix.com/configure-ssh-key-based-authentication-linux/
- https://docs.sqlalchemy.org/en/13/core/engines.html 
- https://www.cyberciti.biz/faq/how-to-use-chmod-and-chown-command/
- https://serverfault.com/questions/128069/how-do-i-prevent-apache-from-serving-the-git-directory
- https://www.guru99.com/postgresql-create-database.html

README file made using Dillinger.io
