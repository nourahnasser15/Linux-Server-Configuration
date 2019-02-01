# Linux-Server-Configuration
 Final Project for Full Stack Web Developber Course part [FSND](https://sa.udacity.com/course/full-stack-web-developer-nanodegree--nd004)
by Norah Aaljlan

This page explains how to secure and set up a Linux distribution on a virtual machine, install and configure a web and database server to host a web application. 

- Linux distribution  [Ubuntu](https://www.ubuntu.com/download/server) 16.04 LTS.
- The virtual private server is [Amazon Lighsail](https://lightsail.aws.amazon.com/).
- My web Application [Item-Catalog-project](https://github.com/nourahnasser15/Item-Catalog)
- The database server is [PostgreSQL](https://www.postgresql.org/).


You can visit http://54.202.88.40/ or http://ec2-54-202-88-40.us-west-2.compute.amazonaws.com for  deployed website.



## Get a server

### Step 1: By useing Instance Amazon Lightsail Start a new Ubuntu Linux server 

- Login into [Amazon Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/resources) using an Amazon Web Services account.


### Step 2: SSH into the server
- download default private key from Amazon Lightsail 
- connect to the instance via the terminal: `ssh -i ~/.ssh/lightsail_key.rsa ubuntu@54.202.88.40`, 
  where `54.202.88.40` is the public IP address of the instance.


## Secure the server

### Step 3: Change the SSH port from 22 to 2200

- Edit the `/etc/ssh/sshd_config` file: `sudo nano /etc/ssh/sshd_config`.
- Change the port number on line 5 from `22` to `2200`.
- Save and exit using CTRL+X and confirm with Y.
- Restart SSH: `sudo service ssh restart`.

### Step 4: Configure the Uncomplicated Firewall (UFW)

- Configure the default firewall for Ubuntu to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
  ```
  sudo ufw status                  # The UFW should be inactive.
  sudo ufw default deny incoming   # Deny any incoming traffic.
  sudo ufw default allow outgoing  # Enable outgoing traffic.
  sudo ufw allow 2200/tcp          # Allow incoming tcp packets on port 2200.
  sudo ufw allow www               # Allow HTTP traffic in.
  sudo ufw allow 123/udp           # Allow incoming udp packets on port 123.
  sudo ufw deny 22                 # Deny tcp and udp packets on port 53.
  ```

- Turn UFW on: `sudo ufw enable`. The output should be like this:
  ```
  Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
  Firewall is active and enabled on system startup
  ```

- Check the status of UFW to list current roles: `sudo ufw status`. The output should be like this:

  ```
  Status: active
  
  To                         Action      From
  --                         ------      ----
  2200/tcp                   ALLOW       Anywhere                  
  80/tcp                     ALLOW       Anywhere                  
  123/udp                    ALLOW       Anywhere                  
  22                         DENY        Anywhere                  
  2200/tcp (v6)              ALLOW       Anywhere (v6)             
  80/tcp (v6)                ALLOW       Anywhere (v6)             
  123/udp (v6)               ALLOW       Anywhere (v6)             
  22 (v6)                    DENY        Anywhere (v6)
  ```

- Exit the SSH connection: `exit`.

- Click on the `Manage` option of the Amazon Lightsail Instance, 
then the `Networking` tab, and then change the firewall configuration to match the internal firewall settings above.

- Allow ports 80(TCP), 123(UDP), and 2200(TCP), and deny the default port 22.

- From your local terminal, run: `ssh -i ~/.ssh/lightsail_key.rsa -p 2200 ubuntu@54.202.88.40`, where `54.202.88.40` is the public IP address of the instance.



### Step 4.1: install `Fail2Ban` to ban attackers 

`Fail2Ban` is an intrusion prevention software framework that protects computer servers from brute-force attacks.
- Install Fail2Ban: `sudo apt-get install fail2ban`.
- Install sendmail for email notice: `sudo apt-get install sendmail iptables-persistent`.
- Create a copy of a file: `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`.
- Change the settings in `/etc/fail2ban/jail.local` file:
  ```
  set bantime = 1800
  destemail = useremail@domain (nourahnasser1@gmail.com)
  action = %(action_mwl)s 
  ```
- Under `[sshd]` change `port = ssh` by `port = 2200`.
- Restart the service: `sudo service fail2ban restart`.


### Step 4.2: Automatically install updates

- Enable automatic (security) updates: `sudo apt-get install unattended-upgrades`.
- Edit `/etc/apt/apt.conf.d/50unattended-upgrades`, uncomment the line `${distro_id}:${distro_codename}-updates` and save it.
- Modify `/etc/apt/apt.conf.d/20auto-upgrades` file so that the upgrades are downloaded and installed every day:
  ```
  APT::Periodic::Update-Package-Lists "1";
  APT::Periodic::Download-Upgradeable-Packages "1";
  APT::Periodic::AutocleanInterval "7";
  APT::Periodic::Unattended-Upgrade "1";
  ```
- Enable it: `sudo dpkg-reconfigure --priority=low unattended-upgrades`.
- Restart Apache: `sudo service apache2 restart`.

### Step 4.3: Updated packages to most recent versions


- I did these commands:
  ```
  sudo apt-get update
  sudo apt-get dist-upgrade
  sudo shutdown -r now
  ```

## Give `grader1` access


### Step 5: Create a new user account named `grader1`

- While logged in as `ubuntu`, add user: `sudo adduser grader1`. 
- Enter a password (twice) and fill out information for this new user.


### Step 6: Give `grader1` the permission to sudo

- by adding a new file under the suoders directory:
```
$ sudo nano /etc/sudoers.d/grader1
```
In the file put in:
```
grader1 ALL=(ALL:ALL) ALL
  ```

then save and quit

- Verify that `grader1` has sudo permissions. Run `su - grader1`, enter the password, 
run `sudo -l` and enter the password again. 



### Step 7: Create an SSH key pair for `grader1` using the `ssh-keygen` tool

- On the local machine:
  - Run `ssh-keygen`
  - Enter file in which to save the key (I gave the name `grader_key`) in the local directory `~/.ssh`
  - Enter in a passphrase twice. Two files will be generated (  `~/.ssh/grader_key` and `~/.ssh/grader_key.pub`)
  - Run `cat ~/.ssh/grader_key.pub` and copy the contents of the file
  - Log in to the grader1's virtual machine
  

- On the grader1's virtual machine:
  - Create a new directory called `~/.ssh` (`mkdir .ssh`)
  - Run `sudo nano ~/.ssh/authorized_keys` and paste the content into this file, save and exit
  - Give the permissions: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
  - Check in `/etc/ssh/sshd_config` file if `PasswordAuthentication` is set to `no`
  - Restart SSH: `sudo service ssh restart`
- On the local machine, run: 
`  ssh -i ~/.ssh/grader_key -p 2200 grader1@54.202.88.40
Enter passphrase for key:123456

`
<!--
Public IP address is 54.202.88.40
ssh -i ~/.ssh/grader_key -p 2200 grader1@54.202.88.40
passphrase for key:123456
-->


## Prepare to deploy the project

### Step 8: Configure the local timezone to UTC

- While logged in as `grader1`, configure the time zone: `sudo dpkg-reconfigure tzdata`.
-choose Asia Then Riyadh



### Step 10: Install and configure Apache to serve a Python mod_wsgi application

- While logged in as `grader1`, install Apache: `sudo apt-get install apache2`.
- Enter public IP of the Amazon Lightsail instance into browser. you should see default page for apache

- install mod_wsgi package:  
 `$ sudo apt-get install libapache2-mod-wsgi python-dev`.
- Enable `mod_wsgi` using: `sudo a2enmod wsgi`.


### Step 11: Install and configure PostgreSQL

- While logged in as `grader1`, install PostgreSQL:
 `sudo apt-get install postgresql`.
- PostgreSQL should not allow remote connections. In the  `/etc/postgresql/9.5/main/pg_hba.conf` file, you should see:
  ```
  local   all             postgres                                peer
  local   all             all                                     peer
  host    all             all             127.0.0.1/32            md5
  host    all             all             ::1/128                 md5
  ```

- Switch to the `postgres` user: `sudo su - postgres`.
- Open PostgreSQL interactive terminal with `psql`.
- Create the `catalog` user with a password and give them the ability to create databases:
  ```
  postgres=# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
  postgres=# ALTER ROLE catalog CREATEDB;
  ```

- List the existing roles: `\du`. The output should be like this:
  ```
                                     List of roles
   Role name |                         Attributes                         | Member of 
  -----------+------------------------------------------------------------+-----------
   catalog   | Create DB                                                  | {}
   postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
  ```

- Exit psql: `\q`.
- Switch back to the `grader1` user: `exit`.
- Create a new Linux user called `catalog`: `sudo adduser catalog`. Enter password and fill out information.
- Give to `catalog` user the permission to sudo
- adding a new file under the suoders directory:
`$ sudo nano /etc/sudoers.d/catalog` 
 In the file put in: `catalog ALL=(ALL:ALL) ALL,`
  then save and quit.

- Save and exit using CTRL+X and confirm with Y.
- Verify that `catalog` has sudo permissions. Run `su - catalog`, enter the password, run `sudo -l` and enter the password again.


- While logged in as `catalog`, create a database: `createdb catalog`.
- Run `psql` and then run `\l` to see that the new database has been created. The output should be like this:
  ```
                                    List of databases
     Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
  -----------+----------+----------+-------------+-------------+-----------------------
   catalog   | catalog  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
   postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
   template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres
   template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres
  (4 rows)
  ```
- Exit psql: `\q`.
- Switch back to the `grader1` user: `exit`.




### Step 11: Install git

- While logged in as `grader1`, install `git`: `sudo apt-get install git`.

## Deploy the Item Catalog project

### Step 12.1: Clone and setup the Item Catalog project from the GitHub repository 

- While logged in as `grader1`, create `/var/www/catalog/` directory.
- Change to that directory and clone the catalog project:
`sudo git clone https://github.com/nourahnasser15/Item-Catalog.git catalog`.
- From the `/var/www` directory, change the ownership of the `catalog` directory to `grader1` using: `sudo chown -R grader1:grader1 catalog/`.
- Change to the `/var/www/catalog/catalog` directory.
- Rename the `FinalProject.py` file to `__init__.py` using: `mv FinalProject.py __init__.py`.

- In `__init__.py`, replace :
  ```
  # app.debug = True
  #app.run(host='0.0.0.0', port=5000)
  
  app.run()
  ```
 and 
   ```
   # engine = create_engine("sqlite:///catalog.db")
   engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
   ``` 


- In `database_etup.py`
   ```
   # engine = create_engine("sqlite:///catalog.db")
   engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
   ``` 

### Step 12.2: Authenticate login through Google

- Go to [Google Cloud Plateform](https://console.cloud.google.com/).
- Click `APIs & services` on left menu.
- Click `Credentials`.
- Create an OAuth Client ID (under the Credentials tab), and add http://54.202.88.40 and 
http://ec2-54-202-88-40.us-west-2.compute.amazonaws.com as authorized JavaScript 
origins.
- Add http://ec2-54-202-88-40.us-west-2.compute.amazonaws.com/login 
as authorized redirect URI.
- Download the corresponding JSON file, open it et copy the contents.
- Open `/var/www/catalog/catalog/client_secret.json` and paste the previous contents into the this file.
- Replace the client ID  of the `templates/login.html` file in the project directory.



### Step 13.1: Install the virtual environment and dependencies

- While logged in as `grader1`, install pip: `sudo apt-get install python-pip`.
- Install the virtual environment: `sudo apt-get install python-virtualenv`
- Change to the `/var/www/catalog/catalog/` directory.
- Create the virtual environment: `sudo virtualenv -p python venv`.
- Change the ownership to `grader1` with: `sudo chown -R grader1:grader1 venv/`.
- Activate the new environment: `. venv/bin/activate`.
- Install the following dependencies:
  ```
  pip install httplib2
  pip install requests
  pip install --upgrade oauth2client
  pip install sqlalchemy
  pip install flask
  sudo apt-get install libpq-dev
  pip install psycopg2
  ```

- Run `python __init__.py` .

- Deactivate the virtual environment: `deactivate`.


### Step 13.2: Set up and enable a virtual host

- Create `/etc/apache2/sites-available/catalog.conf` and add the 
following lines to configure the virtual host:

```
<VirtualHost *:80>
    ServerName 54.202.88.40
   ServerAlias ec2-54-202-88-40.us-west-2.compute.amazonaws.com
   ServerAdmin grader1@54.202.88.40
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
```

- Enable virtual host: `sudo a2ensite catalog`

- Reload Apache: `sudo service apache2 reload`.



### Step 13.3: Set up the Flask application

- Create `/var/www/catalog/catalog.wsgi` file add the following lines:

```
  activate_this = '/var/www/catalog/catalog/venv/bin/activate_this.py'
  with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)

sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = "..."
  ```

- Restart Apache: `sudo service apache2 restart`.


### Step 13.5: Disable the default Apache site

- Disable the default Apache site: `sudo a2dissite 000-default.conf`. 
The following prompt will be returned:

  ```
  Site 000-default disabled.
  To activate the new configuration, you need to run:
    service apache2 reload
  ```

- Reload Apache: `sudo service apache2 reload`.

### Step 13.6: Launch the Web Application

- Restart Apache again: `sudo service apache2 restart`.
- Open your browser to http://54.202.88.40 or http://ec2-54-202-88-40.us-west-2.compute.amazonaws.com


