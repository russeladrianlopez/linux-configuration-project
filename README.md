# Project: Linux Configuration

A walkthrough configuration of ubuntu server instance using Amazon Lightsail, securing the server ports and deploying a Python Flask [Item Catalog Application](https://github.com/russeladrianlopez/item-catalog) with the use of apache2.

* IP Address: [54.144.28.202](http://54.144.28.202)
* SSH Port: `2200`

## Configuration
### Get your server.
##### 1. Start a new Ubuntu Linux server instance on Amazon Lightsail.
* Click the link [here](https://lightsail.aws.amazon.com/). Log in to Lightsail. If you don't already have an Amazon Web Services account, you'll be prompted to create one.
* Once you're logged in, you will be prompted to create an instance. start creating the instance.
* First, choose **"OS Only"** (instead of "Apps + OS"). Then, choose Ubuntu as the operating System.
* Choose your instance plan. I selected the lowest priced one.
* Give instance a hostname.
* Wait for it to start up. _(It may take a few minutes for your instance to start up.)_
##### 2. Follow the instructions provided to SSH into your server.
* Go to your Amazon Lightsail **'Account page'**, then proceed to SSH Keys tab.
* Download the default key pair provided.
* a pem file will be downloaded. move and rename the file in your local ~/.ssh/directory
* provide security by running `chmod 600 ~/.ssh/your_key`
* Login your server using `ssh ~i ~/.ssh/your_key ubuntu@publicip`

### Secure your server.
##### 3. Update all currently installed packages.
* `sudo apt-get update`
* `sudo apt-get upgrade`
* `sudo apt=get autoremove`
##### 4. Change the SSH port from 22 to 2200. Make sure to configure the Lightsail firewall to allow it.
* Go back to Lightsail instance in the browser.
* Go to Networking tab, and Add **Custom TCP 2200** and **Custom UDP 123**.
* `sudo nano /etc/ssh/sshd_config`
* change `Port 22` with Port `2200` instead.
* `sudo service ssh restart`
##### 5. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
###### Start configuring the firewall by running the following:
* `sudo ufw default deny incoming`
* `sudo ufw default allow outgoing`
* `sudo ufw allow ssh`
* `sudo ufw allow www`
* `sudo ufw allow 2200/tcp`
* `sudo ufw allow 80/tcp`
* `sudo ufw allow 123/udp`
* `sudo ufw enable`

### Give grader access.
##### 6. Create a new user account named grader.
* Run `sudo adduser grader` and fill out the details.
##### 7. Give grader the permission to sudo.
* Run `sudo nano /etc/sudoers.d/grader`
* type in `grader ALL=(ALL) NOPASSWD:ALL`
##### 8. Create an SSH key pair for grader using the ssh-keygen tool.
* Run `ssh keygen` in your local machine
* Create a file name for the key pair. Ex.(LinuxKeys) would generate LinuxKeys file and a LinuxKeys.pub
* Go into grader's home directory. create a .ssh directory: `mkdir .ssh`
* Create a authorized_keys file : `sudo nano .ssh/authorized_keys
* Copy contents of the pub file from your local key pair into the authorized_keys file.
* Make sure it is secure by Running: `chmod 700 .ssh/` and `chmod 644 .ssh/authorized_keys` respectively.
* Force key-based authentication by Running `sudo nano /etc/ssh/sshd_config`
* Find and Change `PasswordAuthentication yes` to `PasswordAuthentication no`
* `sudo service ssh restart`
* You may log on the server using `ssh grader@54.144.28.202 -p 2200 -i ~/.ssh/key_name_here`

### Prepare to deploy your project.
##### 9. Configure the local timezone to UTC.
* Run `sudo dpkg-reconfigure tzdata`
* Choose none of above, then UTC.
##### 10. Install and configure Apache to serve a Python mod_wsgi application.
* Run `sudo apt-get install apache2`
* Run `sudo apt-get install libapache2-mod-wsgi`
##### 11. Install and configure PostgreSQL:
* Run `sudo apt-get install postresql`
###### Create a new database user named catalog that has limited permissions to your catalog application database.
* Run `sudo -u postgres createuser -D -A -P catalog`
* Run `sudo -u postgres createdb -O catalog catalog`
##### 12. Install git.
* `sudo apt-get install git`

### Deploy the Item Catalog project.
##### 13. Clone and setup your Item Catalog project from the Github repository you created earlier in this Nanodegree program.
* `cd /var/www/`
* `sudo git clone https://github.com/russeladrianlopez/item-catalog.git`
##### 14. Set it up in your server so that it functions correctly when visiting your server’s IP address in a browser.
###### installing requirements for application
* `sudo apt-get install python-pip`
* `pip install --upgrade pip`
* `sudo pip install flask packaging oauth2client redis passlib flask-httpauth`
* `sudo pip install sqlalchemy flask-sqlalchemy psycopg2 bleach requests`
###### Make sure that your .git directory is not publicly accessible via a browser!
* `sudo nano item-catalog/.htaccess` , then type in `RedirectMatch 404 /\.git`
###### Re-configure application's database connections from sqlite to postgresql
* configure old sqlite database to `python engine = create_engine('postgresql://catalog:iwillpassyou@localhost/catalog')` across all files.
* Once done, Run `python db_setup.py`
###### Re-configure oauth in all files
* follow readme on the original repo [here](https://github.com/russeladrianlopez/item-catalog).
###### Setup WSGI for catalog application
* Create a wsgi file inside app directiory : `sudo nano application.wsgi`
* paste the following:
```
# WSGI file

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/item-catalog')

from application import app as application
application.secret_key='super_secret_key'
```

* Create a wsgi virtual host file : `sudo nano /etc/apache2/sites-available/item-calatog.conf`
* paste the following:
```
# Virtual Host file
<VirtualHost *:80>
     ServerName  54.144.28.202
     ServerAdmin lopez.russeladrian@gmail.com
     # Location of the items-catalog WSGI file
     WSGIScriptAlias / /var/www/item-catalog/application.wsgi
     # Allow Apache to serve the WSGI app
     <Directory /var/www/item-catalog>
          Order allow,deny
          Allow from all
     </Directory>
     # Allow Apache to deploy static content
     <Directory /var/www/item-catalog/static>
        Order allow,deny
        Allow from all
     </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

* `sudo a2dissite 000-default.conf`
* `sudo a2ensite item-catalog.conf`
* `sudo service apache2 restart`

## Run it on your Browser
*  It should be running all fine at this url: http://54.144.28.202
## Tools and Sources
* [Amazon Lightsail](https://lightsail.aws.amazon.com/)
* [Google APIs](https://console.developers.google.com/apis/credentials)
* [Changing Ports in Your Linux Sever](https://www.godaddy.com/help/changing-the-ssh-port-for-your-linux-server-7306)
* [PostgreSQL Ubuntu Community Guidelines](https://help.ubuntu.com/community/PostgreSQL)
* [Flash Guidelines for Apache mod_wsgi](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/#configuring-apache)
* [How To Serve Flask Applications with uWSGI and Nginx on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uwsgi-and-nginx-on-ubuntu-16-04)
* [Ubuntu Documentation for SSH/Keys](https://help.ubuntu.com/community/SSH/OpenSSH/Keys)

Nanodegree Course courtesy of [Udacity](https://www.udacity.com/).
