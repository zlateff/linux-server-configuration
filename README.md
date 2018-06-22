# Linux Server Configuration

This is a [Udacity FSND](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004) project.

## About

This project involves taking a baseline installation of a Linux distribution on a virtual machine and preparing it to host web applications, installing updates, securing it from a number of attack vectors and installing/configuring web and database servers. The application deployed on the server is the [item-catalog-application](https://github.com/zlateff/item-catalog-application).

## Server Information

* IP address: 18.222.81.58
* Application URL: [http://ec2-18-222-81-58.us-east-2.compute.amazonaws.com](http://ec2-18-222-81-58.us-east-2.compute.amazonaws.com)

## Summary of Configuration Changes

#### 1. Update all currently installed packages:
* `$ sudo apt-get update`
* `$ sudo apt-get upgrade`
#### 2. Change the SSH port to a non-default one (2200 per project requirements)
* `$ sudo nano /etc/ssh/sshd_config` and update line `Port 22` to `Port 2200`
* `$ sudo service ssh restart`
* `$ exit` the VM and SSH with the new port to confirm the change:
    - `$ ssh -i THE-PRIVATE-KEY ubuntu@18.222.81.58 -p 2200`
#### 3. Enforce key-based SSH authentication and disable root remote login
* `$ sudo nano /etc/ssh/sshd_config` to update the following lines:
    - `PasswordAuthentication no`
    - `PermitRootLogin no`
* `$ sudo service ssh restart`
#### 4. Configure the Uncomplicated Firewall (UFW)
* Per project requirements, only SSH(2200/tcp), HTTP(80/tcp), and NTP(123/udp) ports should be enabled.
    - `$ sudo ufw default deny incoming`
    - `$ sudo ufw default allow outgoing`
    - `$ sudo ufw allow 2200/tcp`
    - `$ sudo ufw allow www`
    - `$ sudo ufw allow ntp`
    - `$ sudo ufw enable`
* Confirm with `$ sudo ufw status`
* Since using AWS EC2, the [instance security group](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html#security-group-rules) inbound rules must be updated too.
#### 5. Configure the timezone to UTC
* `$ sudo timedatectl set-timezone Etc/UTC`
For automatically keeping the server time accurate:
* `$ sudo apt-get install ntp` per [UbuntuTime](https://help.ubuntu.com/community/UbuntuTime)
#### 6. Add new user and give it sudo privileges
* Per requirements, create user `grader` with `$ sudo adduser grader`
* Create `grader` file in the sudoers directory `$ sudo nano /etc/sudoers.d/grader`
* Add `grader ALL=(ALL) NOPASSWD:ALL` line to the file and save it
#### 7. Configure key-based authentication for the new user
* Generate encryption key pair on local machine using `ssh-keygen`
* On VM, `$ sudo su grader` followed by `$ touch /home/grader/.ssh/authorized_keys`
* `$ nano /home/grader/.ssh/authorized_keys` and copy the contents of the PUBLIC KEY that was generated locally
* Change the `.ssh` and `authorized_keys` file permissions:
    - `$ sudo chmod 700 /home/grader/.ssh`
    - `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`
* To confirm `$ exit` the VM and connect again with `$ ssh -i THE-GRADER-PRIVATE-KEY grader@18.222.81.58 -p 2200`
#### 8. Install Apache and mod_wsgi
* Apache Server - `$ sudo apt-get install apache2`
* mod_wsgi module - `$ sudo apt-get install libapache2-mod-wsgi`
#### 9. Clone and configure the Item Catalog application
* `$ cd /var/www/html` and clone the application files inside `/var/www/html/libraryapp`
* `$ nano /var/www/html/libraryapp/myapp.wsgi` and update the file:
```
import sys
sys.path.insert(0, '/var/www/html/libraryapp')

from app import app as application
```
* `$ sudo nano /etc/apache2/sites-enabled/000-default.conf` and update file:
```
<VirtualHost *:80>
...
  DocumentRoot /var/www/html
  
  WSGIDaemonProcess app threads=5
  WSGIScriptAlias / /var/www/html/libraryapp/myapp.wsgi
  <Directory libraryapp>
    WSGIProcessGroup libraryapp
    WSGIApplicationGroup %{GLOBAL}
    Order deny,allow
    Allow from all
  </Directory>
...
</VirtualHost>
```
#### 10. Install and configure the database:
* Install PostgreSQL - `$ sudo apt-get install postgresql`
* Create a database user named 'catalog' - `$ sudo -u postgres createuser catalog`
* Create a database for the application named 'library' - `$ sudo -u postgres createdb library`
* Connect to the psql terminal with `$ sudo -u postgres psql`
* Give the new user a password `psql=# alter user catalog with encrypted password '<password>';`
* Grant the user privileges on the new DB `psql=# grant all privileges on database library to catalog;`
* Inside the Flask application, set the DB connections using:
    - `engine = create_engine('postgresql://catalog:<password>@localhost/library')`
* Install SQLAlchemy and psycopg2:
    - `$ sudo apt-get install python-sqlalchemy`
    - `$ sudo apt-get install python-psycopg2`
* Setup the application DB and populate with initial data:
    - `$ python /var/www/html/libraryapp/database_setup.py`
    - `$ python /var/www/html/libraryapp/init_database.py`
#### 11. Install all necessary packages for running the application, update API settings and restart Apache
* Update OAuth redirect_uris and javascript_origins in the `client_secrets.json` file with the new application URL.
* Verify and add the domain in the Google API console 
* `$ sudo apt-get install python-flask python-oauth2client python-requests`
* `$ sudo /etc/init.d/apache2 restart`

## Resources consulted
[Ubuntu Packages](https://packages.ubuntu.com/), [Ask Ubuntu](https://askubuntu.com/), [PostgreSQL Docs](https://www.postgresql.org/docs/)