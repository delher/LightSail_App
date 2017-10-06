# LightSail_App
Deployment of a Python app on an Amazon LightSail Ubuntu Linux server

## Establish Account and Ubuntu Instance
1. Establish account at lightsail.aws.amazon.com.
2. Click “create instance.”
3. Make sure location is appropriate (Ohio, Zone A, us-east-2am)
4. Under “pick your instance image,” select “OS Only” tab and then click Ubuntu. The SSH key pair should indicate “default”.
5. Select the Instance Plan (5 USD/mo)
6. Enter a unique name for the instance. (VintageValues) and click the Create button.
7. Wait for the instance to be created. Note the IP address given: 1.2.3.4 is used here as an example.
8. Connect to the instance using the orange >_ icon in the upper right or click on the column of three dots and select “CONNECT.”
9. After confirming the instance is running and responding, close the connection window.

## Private Key Setup for SSH
From the Account/SSH Keys page, download your private key:
https://lightsail.aws.amazon.com/ls/webapp/account/keys

You can rename the key (LS_key.pem is used here) and move it to a convenient directory, such as the local machine's user root directory (cd ~).
Change the permissions on the key file to exclude viewing by other users:
`chmod 400 LS_key.pem`

Test the key and ssh connection on the ssh default port. If your key file is not in the user root directory, include the path to it.
`ssh - i </path/>LS_key.pem ubuntu@1.1.1.1`

## Port Setup
Open the 123 (NTP) and 2200 (SSH) ports for incoming data in the Lightsail configuration
https://lightsail.aws.amazon.com/ls/webapp/us-east-2/instances/InstanceName/networking
(3-dot menu > Manage > Networking tab)
Open a port with TCP protocol at ports 123 and 2200.

## Configure SSH
### Connect at Port 2200 Instead of Port 22
Edit the sshd_config file:
`sudo nano /etc/ssh/sshd_config`

Change the listening port from 22 to 2200. If there is a # before "Port," remove it.

### Disable Password Connections
If the ssh config file shows "ChallengeResponseAuthentication yes" or "PasswordAuthentication yes" change "yes" to "no."

### Disable Remote Login as Root
Change the line under "Authentication" that says "PermitRootLogin" from "yes" or "prohibit-password" to "no"
Save the file (^o), then exit (^x). Restart the ssh service:
`sudo service ssh restart`

### Test the Connection on Port 2200
In a new ssh window, try logging in again. The server should now refuse a connection on the default ssh port. Specify the port for ssh:
`ssh -i LS_key.pem ubuntu@ 1.1.1.1 -p 2200`

Test the connection before removing the SSH port 22 in the Lightsail configuration page:
https://lightsail.aws.amazon.com/ls/webapp/us-east-2/instances/InstanceName/networking
When you remove the port 22 configuration, it will break your SSH pipe, so you'll have to reconnect using your ssh client, designating ssh at port 2200. The "Connect using SSH" option on the Lightsail instance page will no longer work.

`ssh - i LS_key.pem -p 2200 ubuntu@1.2.3.4`

## Activate the Ubuntu Linux firewall

### Make sure the firewall is inactive:
`sudo ufw status`

### Configure the UFW firewall:
```
sudo ufw deny 22
sudo ufw allow ssh
sudo ufw allow www
sudo ufw allow ntp
sudo ufw allow 2200/tcp
sudo ufw allow 123/tcp
```

### CONFIRM YOU HAVE SSH ACCESS ON PORT 2200 BEFORE PROCEEDING.
### Activate the firewall:
`sudo ufw enable`

## Update/upgrade the Ubuntu installation:
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```
Restart the instance using the 3-dot menu:
https://lightsail.aws.amazon.com/ls/webapp/home/resources

#Add “grader” as a new user:
`sudo adduser grader`

Enter a password for grader
Record the password - you’ll need it later.
Confirm the password by retyping it.
Enter name information for user grader: A Grader
User grader should now be the last entry in the list displayed by
`sudo cat /etc/passwd`

## Give grader sudo permission

`sudo usermod -aG sudo grader`

### Make sure grader has sudo privileges by changing user:
`su - grader`
### Enter the grader password, then enter
`sudo ls -la /root`
Enter the grader password again as requested.
If the sudo privilege is configured correctly, you’ll see the /root directory contents.
Reference: https://www.digitalocean.com/community/tutorials/how-to-create-a-sudo-user-on-ubuntu-quickstart


## Create Grader's Private Key

### Generate an SSH Key Pair for Grader on your local machine:
`ssh-keygen`
Indicate where to save the key file: /Users/yourname/grader_key
Enter a passphrase if you'd like one: abracadabra
Verify the passphrase by re-entering it. SSH-KeyGen will create two files: a private key file named grader_key and a public key file, grader_key.pub.

### Change the permissions for the private key file so it is read only, and other users cannot read or write to the file:
`chmod 400 grader_key`

### open the .pub file using the "less" file reader and copy the key sequence.
`less grader_key.pub`
select the key sequence, ^C to copy, then q to exit the less reader

### Connect to your instance, then change user to grader and move to the grader's home directory:
```
su grader
cd ~
mkdir .ssh
cd .ssh
touch authorized_keys
nano auauthorized_keys
```
Paste in the grader .pub key sequence, write the file (^o) and exit (^x)

Type exit to return to your user, then logout.
Set the permissions on the grader private key file:
`chmod 400 grader_key`

### Confirm Grader Login
Try to log in as grader using the grader's private key:
`ssh -i grader_key -p 2200 grader@1.2.3.4`
Enter the passphrase if you set one.

If the setup is correct, you'll be logged into your instance as "grader."
Close the connection:
`logout`

## Select UTC as the time zone for this instance:
`sudo dpkg-reconfigure tzdata`
scroll down the list to select "None of the Above", then select UTC and exit. A confirmation message will show the current time zone.

## Apache Server Setup

`sudo apt-get install apache2`

Check for a successful installation by visiting your instance IP address in your browser:
http://1.2.3.4
A successful install of the Apache server will show an Ubuntu/Apache2 "It Works" page.

# Install Python mod_wsgi
`sudo apt-get install libapache2-mod-wsgi`

# Test a Python-Flask App:
## Create a backup directory, then save the default configuration file as a backup:
```
cd /etc/apache2/sites-enabled
mkdir backup
sudo cp 000-default.conf backup/000-default-backup.conf
```

### Edit the configuration file:
'sudo nano 000-default.conf`

Add this line just before </VirtualHost>
`WSGIScriptAlias / /var/www/html/myapp.wsgi`

Save and exit nano, then restart the Apache server:
`sudo service apache2 restart`

### Test the Cofiguration
Create a test Python app in the path pointed to by the 000-default.conf file:

`sudo nano /var/www/html/myapp.wsgi`
Enter:
```
  def application(environ, start_response):
        status = '200 OK'
        output = 'Hello World!'
        response_headers (('Content-type', 'text/plain'), (Content-Length, str(len(output))))
        start_response(status, response_headers)
        return [output]
 ```
The WSGI modification should return the "Hello World!" response.

## Install Supporting Apps
### PostgreSQL (current version = 9.5)

`sudo apt-get install postgresql postgresql-contrib`

The PostgreSQL config file is configured by default to ignore remote connections. The "host" entries should have only 127.0.0.1 (localhost IPV4) or ::1/128 (loopback IPv6)
change the "local" entry from peer to md5 to allow connections.
`sudo nano /etc/postgresql/9.5/main/pg_hba.conf`

Restart PostgreSQL:
`sudo service postgresql restart`

### Set up the PostgreSQL Server and Configure a Password:
```
sudo -u postgres psql postgres
postgres=# \password postgres
Enter a PostgreSQL password at the prompt and again to confirm it.
Enter \q to exit from the PostgreSQL prompt.
```
### Create a new user for Postgres
```
sudo su - postgres
psql
CREATE USER catalog WITH PASSWORD 'YourNewUserPassword'
```

### Check the User Entry
`\du`

### Limit the New User's Permissions
```
ALTER ROLE catalog WITH LOGIN;
ALTER USER catalog CREATEDB;
```
### Create the Database:
```
CREATE DATABASE your_database_name WITH OWNER catalog;
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
```
\q to quit, then 'exit' to change back from postgres user to your username.

### Restart the Database
`sudo service postgresql restart`

Database Setup Resources:
http://suite.opengeo.org/docs/latest/dataadmin/pgGettingStarted/firstconnect.html
https://www.postgresql.org/docs/9.4/static/auth-pg-hba-conf.html

## Git Configuration
Git is preinstalled. To configure it, enter:
```
git config --global user.name "Firstname Lastname"
git config --global user.email "name@domain.com"
cd /var/www
mkdir catalog
cd catalog
sudo git clone https://github.com/git_user_name/git_repo_name.git
```
Your git repo is copied to /var/www/catalog/vv

### Prevent www access to Git repo
`sudo nano/etc/apache/conf-enabled/security.conf`
Add these lines after the commented example on "DirectoryMatch":
```
<DirectoryMatch "/\.git">
Require all denied
</DirectoryMatch>
```
Restart the web server
`sudo service apache2 restart`

Check to see if a file such as .git can be accessed by a web browser:
`http://1.2.3.4/catalog/vv/.git/config.git`

Resource: http://dev-notes.eu/2017/01/apache-directives-in-config-vs-htaccess/

## Pip and Python Installation
### Install Pip
sudo apt-get install python-pip
pip install --upgrade pip

### Create Virtual Environment for Python
sudo virtualenv venv

### Install Flask in Virtual Environment
`source venv/bin/activate`

At the (venv) prompt, enter
`sudo pip install Flask`
and install any other dependencies your app may have. For example:
```
sudo pip install requests
sudo pip install random
sudo pip install json
sudo pip install sqlalchemy
sudo pip install psycopg2
sudo pip install oauth2client
sudo pip install httplib2
```
Make a small Flask test app:
`sudo nano __init__.py`
and enter
```
from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello():
    return "Hello, from Flask in the virtual environment!"
if __name__ == "__main__":
    app.run()
```
Save the file and exit nano.

`sudo python __init__.py`
should return
"Running on http://127.0.0.1:5000"
Hit ctrl-C to exit the virtual environment, then
`deactivate`
to deactivate it.

Add a conf file with 'ServerName' specified as your LightSail URL:
`sudo nano /etc/apache2/sites-available/YourApp.conf`
with the contents:
```
<VirtualHost *:80>
                ServerName ec2-18-221-215-100.us-east-2.compute.amazonaws.com
                ServerAdmin admin@mywebsite.com
                WSGIScriptAlias / /var/www/YourApp.wsgi
                <Directory /var/www/catalog/>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/static
                <Directory /var/www/catalog/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### Create .wsgi File
`sudo nano YourApp.wsgi`

Add the following:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/YourApp/")
```
from YourApp import app as application
application.secret_key = 'Add your secret key''

### Modifications to Python Code and Database
If the original application code uses SQLite, the SQLchemy engine must be replaced so it uses PostgreSQL:
Replace:
`engine = create_engine('sqlite:///...')`
with
`engine = create_engine('postgresql://YourDataBaseUserName:YourDatabaseUserPassword@localhost/DatabaseName')`

Change any DATETIME entries used in your database creation script with DateTime; replace DATETIME import from SQLAlchemy.dialects.sqlite with DateTime from sqlalchemy.

### Python/Flask Resource
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps

## Update Google's OAuth Information
If your app allows a Google login, add your instance IP and URLs for Javascript origins and redirects at
https://console.developers.google.com/. The URL can be built from the IP and server location using this example:

http://ec2-18-221-215-100.us-east-2.compute.amazonaws.com/

## Restart Web Server
Restart the www server after any changes to the .conf or .wsgi files:
`sudo service apache2 restart'

## Troubleshooting
### Error Log
If the server is not working, check the Apache server error log:
`less /var/log/apache2/error.log`


















