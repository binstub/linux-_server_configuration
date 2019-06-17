# Linux Server Configuration    
This project is to configure a Linux virtual machine so we can publish [Item Catalog](https://github.com/binstub/item_catalog).

- IP Address : 3.1.204.125
- SSH Port : 2200
- Host Name : 3.1.204.125.xip.io
+ URL to Item Catelog : http://3.1.204.125/catalog

## Task 1 - Get Your Server
1. Start a new Ubuntu Linux server instance on [Amazon Lightsail](https://aws.amazon.com/lightsail/)

2. Choose OS Only and Ubuntu Server. Free Plan is sufficient for this project.

3. SSH into your new server based on your host machine.
    + Download the default private key and rename it, say LightsailDefaultKey.pem
    + Move LightsailDefaultKey.pem to the folder ~/.ssh/ in your local machine
    + Set file rights `chmod 600 ~/.ssh/LightsailDefaultKey.pem`. Set .ssh folder permission to 700
    + SSH into the new server instance from your local machine
    `ssh -i ~/.ssh/LightsailDefaultKey.pem ubuntu@PUBLIC_IP_ADDRESS`


## Task 2 - Give Grader Access
1. Create a new user account named _grader_.
    + `sudo adduser grader`, enter password of your choosing, fill in fields

2. Give grader the permission to sudo.
    + `sudo touch /etc/sudoers.d/grader`
    + `sudo nano /etc/sudoers.d/grader`,
    + `grader ALL=(ALL:ALL) ALL`, save and quit

3. Create an SSH key pair for grader:
    + On your local machine - use `ssh-keygen` to create a private key, then save it in ~/.ssh on the local machine
    + On your virtual machine - login as grader
    + `su - grader`
    + `mkdir .ssh`
    + `touch .ssh/authorized_keys`
    + `nano .ssh/authorized_keys`
    + Copy the public key generated on your local machine to this file and save
    + `chmod 700 .ssh`
    + `chmod 644 .ssh/authorized_keys`
    + `sudo service ssh restart`
   You an now ssh into server using `ssh -i PRIVATE_KEY_FILENAME grader@PUBLIC_IP_ADDRESS`

## Task 3 - Secure Your Server
1. Run the following commands to update your system:
    +  `sudo apt-get update`
    +  `sudo apt-get upgrade`

2. Change the SSH port from 22 to 2200.
    + Go to your Lightsail instance online and click the "Networking" tab, under the "Firewall" heading, add a 'Custom' TCP port of 2200, save changes
    + Open config file and change SSH port from 22 to 2200
    `sudo nano /etc/ssh/sshd_config`

    `# What ports, IPs and protocols we listen for`
    `Port 2200`
    + Save and quit
    `sudo service ssh restart`

3. Require key based logins and prevent logins as root.
    `sudo nano /etc/ssh/sshd_config`
    + Change these lines from:
    `PermitRootLogin yes` to `PermitRootLogin no`
    + and
    `PasswordAuthentication yes` to `PasswordAuthentication no`
      Save and Quit
    + Reload ssh service
    `sudo service ssh restart`

4. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123). Also update the same in Amazon Light Sail Network Tab.
    + Check the ufw status.
    `sudo ufw status`

    + Allow specific connections.
     `sudo ufw allow 2200/tcp`
     `sudo ufw allow 80/tcp`
     `sudo ufw allow 123/udp`

    + Enable firewall.
    `sudo ufw enable`

## Task 4 - Prepare to Deploy Your Project
1. Configure the local timezone to UTC.
    + Run the following command to set the time to UTC
    `sudo timedatectl set-timezone UTC`
    Run `sudo timedatectl status` to verify the settings.

2. Install and configure Apache to serve a Python mod_wsgi application.
    + Install Apache
    `sudo apt-get install apache2`

    + Start Apache
    `sudo service apache2 start` - verify this typing the IP address:80 in your browser

    + Install mod_wsgi
    `sudo apt-get install libapache2-mod-wsgi python-dev`
        (Note: For Python 3 run:`sudo apt-get install libapache2-mod-wsgi-py3`.

    + Enable mod_wsgi
    `sudo a2enmod wsgi`

    + Configure Apache to handle requests using the WSGI module
    `sudo nano /etc/apache2/sites-enabled/000-default.conf`

    + Add `WSGIScriptAlias / /var/www/catalog/itemcatalog.wsgi` before `</VirtualHost>` closing line

3. Install a Virtual Environment
    + Install pip
    `sudo apt-get install python-pip`

    + Install the Virtual Environment
    + `sudo pip install virtualenv`
    + `sudo virtualenv venv`
    + `sudo chmod -R 777 venv`
    + `source venv/bin/activate`

4. Install Flask (within the virtual environment - you'll see 'venv' before the user in the command line)
    `sudo pip install Flask`

    + Install other needed libraries within VM
    `sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils`

5. Install and configure PostgreSQL.
    + `sudo apt-get install libpq-dev python-dev`
    + `sudo apt-get install postgresql`

    + Switch over to the postgres account and open the Postgres prompt by typing:
     `sudo -u postgres psql`
    + Inside the Postgres prompt, you can create a new database user called `catalog` by typing:
      `CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';`
    + To create a new database named `itemcatalog` run:
      `CREATE DATABASE itemcatalog;`

## Task 5 - Deploy the Item Catalog Project
1. Install git
    `sudo apt-get install git`

2. Install Item Catalog from github
    + Change to the www directory
    `cd /var/www`

    + Make a new directory called catalog
    `sudo mkdir catalog`

    + Change the owner to grader
    `sudo chown -R grader:grader catalog`

    + Change to the new directory
    `cd catalog`

    + Clone the github item-catalog repository with
    `git clone https://github.com/binstub/item_catalog catalog`

    + Create a new catalog.wsgi file in the /var/www/catalog/ directory and open it in nano
    `sudo nano itemcatalog.wsgi`

    + Add the following code:
    ```
    #!usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/catalog")
    from item_catalog import app as application
    ```
    Save the changes and exit     

3.  + Copy your main project file (application.py) into the init.py file `mv application.py __init__.py`
    + Change all references to client_secrets.json as /var/www/catalog/item_catalog/client_secrets.json
    Note: Make sure client_secret.json has update info after adding server to the list

4.  Initialize DB tables
   + Change the all `create_engine` lines to in python files to
     create_engine("postgresql://catalog:catalog@localhost/itemcatalog")
     Run the following commands
     ```
     source venv/bin/activate`
     python database_setup.py
     python populate_database.py
     deactivate
     ```
   + Add the servers domain name to the authorized Javascript origins and the allowed forwarding urls in the Google’s Developer Console.    

5. Configure and enable virtual host
    `sudo nano /etc/apache2/sites-available/catalog.conf`
    and add this code:
    ``` 
    <VirtualHost *:80>
       ServerName 3.1.204.125
       ServerAdmin gbindu08@gmail.com
       ServerAlias 3.1.204.125
       WSGIScriptAlias / /var/www/catalog/itemcatalog.wsgi
       <Directory /var/www/catalog/item_catalog/>
           Order allow,deny
           Allow from all
       </Directory>
   
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```
    Save file and exit

    + Enable the virtual host
    `sudo a2ensite catalog`

5. Make sure that your .git directory is not publicly accessible via a browser.
    + `cd var/www/catalog/`
    + `sudo nano .htaccess`
    + Add `RedirectMatch 404 /\.git`
    + Save file and exit



## Author
 
 Bindu Govindaiah

## References

- https://www.postgresql.org/docs/9.5/auth-pg-hba-conf.html
- https://serverfault.com/questions/262751/update-ubuntu-10-04/262773#262773
- https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
- https://www.digitalocean.com/community/tutorials/how-to-set-up-an-apache-mysql-and-python-lamp-server-without-frameworks-on-ubuntu-14-04
