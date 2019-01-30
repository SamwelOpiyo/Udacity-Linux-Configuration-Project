# Udacity Linux Server Configuration Project

## Connect to server via

IP address : 52.15.171.191

SSH port : 2200

EC2 URL : http://ec2-52-15-171-191.us-east-2a.compute.amazonaws.com/
(This instane is no longer available.Please use it as reference.)

## Configuration steps 
### 1. Create an instance in AWS Lightsail 

Go to AWS Lightsail and create a new account / sign in with your account.

Click `Create instance` and choose `Linux/Unix`, `OS only`, `Ubuntu 16.04LTS`

Choose a payment plan (the cheapest plan is enough for now and it`s free for first month)

Click `Create` button to create an instance.

### 2. Set up your SSH key

Go to `account` page from your AWS account. You will find your SSH key there.

![Alt text](https://screenshotscdn.firefoxusercontent.com/images/9e9fb26d-a16d-4e29-9f50-b3a471dfb13a.png)

Download your public SSH key, `LightsailDefaultKey-*.pem`

Navigate to the directory where your file is stored in your terminal (e.g.`Users/MacUser/.ssh`).

Run `chmod 600 LightsailDefaultKey-*.pem` to restrict  file permissions. 

Change name to `lightsail_key.rsa`.

Run a command `ssh -i lightsail_key.rsa ubuntu@52.15.171.191` in your local terminal.

### 3. Change SSH port 22, to 2200

navigate to `/etc/ssh/sshd_config` and edit by entering `sudo nano /etc/ssh/sshd_config`

Change `port 22` to `port 2200`

Save the change by `Control + X`, `y`. This will save changes, and exit.

Restart SSH with `sudo service ssh restart`

### 4.  Setup UFW (Uncomplicated Firewall).

COnly allow incoming request from port `2200(SSH)`, `port 80 (HTTP)` and `port 123 (NTP)`.

Run the following commands:

`sudo ufw status` -- make sure the status is `inactive`.

`sudo ufw default deny incoming` -- deny all incoming requests

`sudo ufw default deny outgoing`-- deny all outgoing requests

`sudo ufw allow 2200/tcp` -- allow incoming ssh request

`sudo ufw allow 80/tcp` -- allow all http request

`sudo ufw allow 123/udp` -- allow ntp request

`sudo ufw deny 22` -- denies incoming request for `port 22`

`sudo ufw enable` -- enables ufw changes above.

`sudo ufw status` -- check current status of ufw. Which should now be `active`
.
Go to AWS page and set the above relevant `custom` ports from `networking` tab.

### 5. Create a new user called grader and give an access 
Run `sudo adduser grader` to create a new user called `grader`

Create a new directory in sudoer by typing `sudo nano /etc/sudoers.d/grader`

Enter the following into the editor `grader ALL=(ALL:ALL) ALL`.

Run `sudo nano /etc/hosts`

Create an SSH key(s) for a grader user with `ssh-keygen` in your local machine.

Copy the generated SSH to a virtual environment.

Run the following command in your virtual environment.

`su - grader`

`mkdir .ssh`

`touch .ssh/authorized_keys`

`nano .ssh/authorized_keys`. Now copy your generated SSH key here.

Reload SSH with `service ssh restart`

You are now able to login as `grader`.

Disable rootlogin.

cd into `/etc/ssh/sshd_config`, find the entry `PermitRootLogin` and change it to `no`.

### 6. Update all packages 

Run `sudo apt-get update` and `sudo apt-get upgrade`

### 7. Set up local time zone to UTC

Run `sudo dpkg-reconfigure tzdata` 

Select `None of the above` then select `UTC`

### 8. Install Apache application and wsgi module

Run `sudo apt-get install apache2`. This will install Apache 2.

Run `sudo apt-get install python-setuptools libapache2-mod-wsgi` to install mod-wsgi module

Turn the server on with `sudo service apache2 start`

### 9. Installing git

Run `sudo apt-get install git`

Configure your username with  `git config --global user.name <username>` 

Configure your email with `git config --global user.email <email>`

### 10. Clone the project

Run `cd /var/www`

Run `sudo mkdir catalog`

Change the owner to grader `sudo chown -R grader:grader catalog`

Run `sudo chmod catalog` to give permission to clone the project.

Switch to the `catalog` directory and clone the Catalog project (e.g `cd /var/www/catalog`).

`cd catalog` and `git clone https://github.com/oinga/Foodie-Catalog.git`

Add `catalog.wsgi` file.

Run `sudo nano catalog.wsgi` and add the following code.

```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, `/var/www/catalog/Foodie-Catalog`)

from Foodie-Catalog import app as application
application.secret_key = `secret`
```

Modify filenames to deploy on AWS.

Rename `run.py` to `__init__.py` by entering`mv run.py  __init__.py`

### 11. Install virtual environment and Flask framework

To install `pip` run, `sudo apt-get install python-pip`

Run `sudo apt-get install python-virtualenv` to install virtual environment.

Create a new virtual environment with `sudo virtualenv venv` and activate it `source venv/bin/activate`

Change permissions to the virutual environment folder 

Run `sudo chmod -R 777 venv`

Install Flask 
Run `pip install Flask` 

Install program dependencies. `pip install bleach httplib2 request oauth2client sqlalchemy python-psycopg2.`

### 12. Configure Apache

Create an  Apache config file `sudo nano /etc/apache2/sites-available/catalog.conf`

Paste the following code
```
<VirtualHost *:80>
    ServerName 52.15.171.191
    ServerAdmin admin@Public-IP-Address
    WSGIScriptAlias / /var/www/catalog/Foodie-Catalog/catalog.wsgi
    <Directory /var/www/catalog/Foodie-Catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/Foodie-Catalog/static
    <Directory /var/www/catalog/Foodie-Catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Enable the new virtual host `sudo a2ensite catalog`

### 13. Install and configure PostgressSQL

Run `sudo apt-get install PostgreSQL`

Check if no remote connections are allowed with `sudo vim /etc/postgresql/9.3/main/pg_hba.conf`

Login to postgress `sudo su - postgres` and `psql`

Create a new user `createuser catalog --pwprompt`. Enter is `password` for the password.

Create a DB called `catalog` with `ALTER USER catalog CREATEDB` and `CREATE DATABASE catalog WITH OWNER catalog`

Create database named `catalog` for the user `catalog` by running `createdb -O catalog catalog`

Connect to the DB with `psql catalog`

Revoke all rights `REVOKE ALL ON SCHEMA public FROM public`

Change a grand from public to catalog `GRANT ALL ON SCHEMA public TO catalog`

Logout from postgress and return to the grader user ` \q` and `exit`

Change the engine inside Flask application.

`engine = create_engine(`postgresql://catalog:catalog@localhost/catalog`)`

Set up the DB with `python /var/www/catalog/item-catalog-udacity/database_setup.py`

### 14. Restart Apache 

Run `sudo service apache2 restart` and check `http://52.15.171.191/`
