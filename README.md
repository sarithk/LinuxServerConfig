# Linux Server Configuration
A project to set up a Linux server. Item Catalog web application runs live on Amazon Lightsail secure web server. A baseline installation of a Linux distribution on a virtual machine is taken and prepared to host the web application, installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

The item catalog is a RESTful web application that displays an item catalog allowing the user to login and add, manage items they add. It interacts with a sqlite database using SQLAlchemy, an Object-Relational Mapping (ORM) layer. The CRUD (create, read, update and delete) operations and web page templates are handled using Python Flask framework. OAuth 2.0 framework allows users to securely login to the application using Google+ Sign-In or Facebook Login so users can create items that are viewable by everyone but only modifiable by the original creator. This application also provides a JSON endpoint.</br>
AWS URL: http://ec2-52-26-30-44.us-west-2.compute.amazonaws.com</br>
Raw IP: 52.26.30.44</br>
SSH port: 2200</br>
JSON Endpoint: http://52.26.30.44/catalog/JSON


## Steps Followed for Configuration
### 1 -  SSH from local machine to the lightsail virtual server instance
* Download the AWS provided default private key to the local machine</br>
* Change the private key file permission not to be accessible by others `chmod 600 LightsailDefaultPrivateKey-us-west-2.pem`</br>
* Logged in, ssh successful ` ssh ubuntu@52.26.30.44 -p 22 -i ~/Downloads/LightsailDefaultPrivateKey-us-west-2.pem`

### 2 -  Install updates
* Updates the list of available packages and versions for upgrade `sudo apt-get update`</br>
* Installs newer versions of the packages `sudo apt-get upgrade`</br>

### 3 - Create user grader
* Create new user grader `sudo adduser grader`</br>
* Give grader the permission to sudo `sudo touch /etc/sudoers.d/grader`</br>
* Check if grader file was added in sudoers directory `sudo ls /etc/sudoers.d`</br>
* Edit the file grader to add the sudo access `sudo nano /etc/sudoers.d/grader`</br>
   . grader ALL=(ALL) NOPASSWD:ALL</br>
   . Save changes(Ctrl+X, Ctrl+Y, Enter)

### 4 - Create an SSH key pair for grader using the ssh-keygen tool
* On the local machine terminal, generate key pair using `ssh-keygen`, providing passphrase
* Named it linuxServerConfig.pub
* Save the private key in `~/.ssh` on local machine
* On the server instance i.e virtual machine:
    ```
    su - grader
    mkdir .ssh
    touch .ssh/authorized_keys
    nano .ssh/authorized_keys

    Copy the public key content from local machine to this file and save, change the access level
    chmod 700 .ssh
    chmod 644 .ssh/authorized_keys
    ```

### 5 - Change the SSH port from 22 to 2200
* Configure the Lightsail firewall in AWS console to allow port 2200 ssh connections
* Make changes in the config file in the server from port 22 to 2200 ```sudo nano /etc/ssh/sshd_config```
* Restart SSH `sudo service ssh restart`
* Configure the Lightsail firewall in AWS console to allow incoming conections into ports HTTP (port 80), and NTP (port 123)
* Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
    ```
    sudo ufw allow 2200/tcp
    sudo ufw allow www
    sudo ufw allow 123/udp
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw enable
    sudo ufw status
    ```
* SSH into grader using the key on 2200 port `ssh grader@52.26.30.44 -p 2200 -i ~/.ssh/linuxServerConfig`

### 6 - Prepare to deploy the project
  SSH to the server</br>
#### Configure the local timezone to UTC
* Check current timezone setting `sudo cat /etc/timezone`
* Configure the local timezone to UTC `sudo dpkg-reconfigure tzdata`

#### Install and configure Apache to serve a Python mod_wsgi application
* Install mod_wsgi package(although using python3, this serves well without causing errors) `sudo apt-get install libapache2-mod-wsgi`
* Install Apache `sudo apt-get install apache2`
* Once done http://52.26.30.44 serves the default apache page

#### Install and configure PostgreSQL
  `sudo apt-get install postgresql`
* Test switching to postgres user
    ```
    sudo su - postgres
    psql
    \q
    exit
    ```
* Do not allow remote connections
   Whether remote connections are allowed to be made to the database is controlled in the pg_hba.conf file. The default is to only allow connections from localhost
   ```
    sudo ls /etc/postgresql/9.5/main
    sudo cat /etc/postgresql/9.5/main/pg_hba.conf
    sudo service apache2 restart
   ```

* Create a new database user named catalog that has limited permissions to catalog application database
    ```
    sudo su - postgres
    CREATE USER catalog WITH PASSWORD 'catalog';
    \du
    CREATE DATABASE catalog;
    \l
    ```

### 7 - Deploy the Item Catalog app from git and create the required set up on server
* Create the FlaskApp, cloning the git repository
    ```
    cd /var/www
    sudo mkdir FlaskApp
    cd FlaskApp
    sudo git clone https://github.com/sarithk/ItemCatalog.git
    tree
    sudo apt install tree
    tree
    ```

* Since my GitHub did not have the client_secrets.json and fb_client_secrets.json, create the files and copy the contents in them
    `sudo nano client_secrets.json`
    `sudo nano fb_client_secrets.json`
* Make changes for the app in Google API Console and Facebook app to accept the lightsail public IP, URL and redirect URIs. Edit both secret files on server to reflect the changes if any
* Edit the python files to reflect the correct postgresql database connection - username, password, DBname
    `sudo nano projectFinal.py`
    `sudo nano DatabaseSetup.py.py`
    change engine = create_engine('sqlite:///itemcatalog.db') to engine = create_engine('postgresql://catalog:catalog@localhost/catalog')

* Rename projectFinal.py to __init__.py using `sudo mv projectFinal.py __init__.py`

#### Install Flask
Setting up a virtual environment will keep the application and its dependencies isolated from the main system. Changes to it will not affect the cloud server's system configurations.
  #### Install other required modules and run init.py without any errors.

    sudo apt-get install python-pip
    sudo pip install virtualenv
    sudo virtualenv venv
    source venv/bin/activate
    sudo pip install Flask
    pip install httplib2
    sudo apt-get install python3-oauth2client
    sudo apt-get install python3-requests
    sudo apt-get install python-requests
    sudo apt-get install  python3-sqlalchemy
    sudo apt-get python3-psycopg2
    sudo python __init__.py

#### Enable the FlaskApp

    sudo nano /etc/apache2/sites-available/FlaskApp.conf

    <VirtualHost *:80>
        ServerName 52.26.30.44
        ServerAdmin sarithakamath24@gmail.com
        WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
        <Directory /var/www/FlaskApp/ItemCatalog/>
            Order allow,deny
            Allow from all
        </Directory>
        Alias /static /var/www/FlaskApp/ItemCatalog/static
        <Directory /var/www/FlaskApp/ItemCatalog/static/>
            Order allow,deny
            Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        </VirtualHost>

     sudo a2ensite FlaskApp

#### Create the .wsgi File

`cd /var/www/FlaskApp`
`sudo nano flaskapp.wsgi`

Add the following lines of code to the flaskapp.wsgi file

    #!/usr/bin/python3
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/ItemCatalog/")

    from FlaskApp import app as application
    application.secret_key = 'super_secret_key'

#### Restart Apache
`sudo service apache2 restart`
Check error log for errors if any, features not working: `sudo cat /var/log/apache2/error.log`

#### Add tables and data
Run database setup file: `sudo python3 /var/www/FlaskApp/ItemCatalog/DatabaseSetup.py`</br>
Connect to database using : \c catalog</br>
\dt
Update test data: `sudo nano /var/www/FlaskApp/ItemCatalog/TestDataUpload.py` </br>
Run the file: `sudo python3 TestDataUpload.py`</br>
Update data from the website as well


#### Other changes in __init__.py and errors handled:
   Change the reference to DatabaseSetup file import relative to the path that was added in the WSGI file i.e ItemCatalog.DatabaseSetup</br>
   Change the file references for client_secrets to reflect the full path i.e /var/www/FlaskApp/ItemCatalog/client_secrets.json</br>
   CLIENT_ID = json.loads(open(r'/var/www/ItemCatalog/client_secrets.json', 'r').read())['web']['client_id']</br>
   Fix import errors by installing the modules for python3 using apt-get</br>
   Disable the default apache2 site</br>
   `sudo a2dissite 000-default`</br>
   `service apache2 restart`</br>
   Make sure the correct app details reflect in the client_secrets files and the coresponding redirect settings in the app consoles.</br>
   Since the python files have PEP8 formatting, make sure there is no space or newline in URL construction to the API calls</br>

* Resources
Udacity forums</br>
https://help.ubuntu.com/</br>
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps</br>
https://www.postgresql.org/docs/current/static/sql-createuser.html</br>
http://flask.pocoo.org/docs/0.12/</br>















