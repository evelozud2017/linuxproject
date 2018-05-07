# Linux Server Configuration Project
This project aims to deploy the previously developed Item Catalog App in a Linux instance.  Item Catalog App is a simple web application developed using Flask.

# Project Details
- IP address = 34.216.227.195
- URL = http://ec2-34-216-227-195.us-west-2.compute.amazonaws.com/
- ssh port = 2200


# Summary of Software Installed
- Apache2
- mod_wsgi
- PostgreSQL
- git
- pip
- virtualenv
- httplib2
- requests
- oauth2client
- sqlalchemy
- flask
- libpq-dev
- pyscopg2
- passlib


# Summary of Configurations made
### Set-up your server
1. Start a new Ubuntu Linux server instance on [Amazon Lightsail](https://lightsail.aws.amazon.com/)
2. Follow instructions from [Udacity](https://classroom.udacity.com/nanodegrees/nd004/parts/ab002e9a-b26c-43a4-8460-dc4c4b11c379/modules/357367901175462/lessons/3573679011239847/concepts/c4cbd3f2-9adb-45d4-8eaf-b5fc89cc606e)

### Secure your server
3. Update all currently installed packages
`sudo apt-get update`
`sudo apt-get upgrade`

4. Change the SSH port from __22__ to __2200__
`sudo vi /etc/ssh/sshd_config`
Comment or overwrite the line with __Port 22__ to show __Port 2200__
    In the Networking tab of the Lightsail instance, update Firewall by adding a new __Custom__ application with __TCP__ Protocol and Port as __2200__.
Restart ssh `sudo /etc/init.d/ssh restart`
To check if it works, try to ssh using port 2200.

5. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
`sudo ufw status`
Result: __Status: inactive__
`sudo ufw default deny incoming`
`sudo ufw default allow outgoing`
`sudo ufw allow 2200/tcp`
`sudo ufw allow 123/udp`
Run `sudo ufw status numbered` to show the list of rules with number.  
Delete the rule corresponding to port __22__ by deleting the rule number `sudo ufw delete x` - x is rule number


### Give grader access ###

6. Create a new user account named __grader__.
`sudo adduser grader`

7. Give __grader__ the permission to __sudo__.
`sudo nano /etc/sudoers`
Add a line after __root    ALL=(ALL:ALL) ALL__ and add `grader ALL=(ALL) NOPASSWD:ALL`
8. Create an SSH key pair for grader using the ssh-keygen tool.
    * From local machine, run `ssh-keygen -t rsa`.  This will create 2 files __id_rsa.pub__ and __id_rsa__ in the /users/eva/.ssh directort.
    * Copy the public key to server `scp -i /Users/eva/.ssh/fsndkeyfile.pem -P 2200 /users/eva/.ssh/id_rsa.pub ubuntu@xx.xxx.xx.xxx:` This will copy the file to home dir of ubuntu.
    * Create .ssh directory in home directory of grader
    `su grader` - this will prompt for a password.  Enter the password that was set when user was created.
    `cd`
    `mkdir .ssh`
    * Still under grader login, copy the id_rsa.pub to a file named __authorized_keys__ in /home/grader/.ssh
    `cd /home/grader`
    `cat /home/ubuntu/id_rsa.pub > .ssh/authorized_keys`
    * Modify permissions of .ssh
    `chmod 600 .ssh/*` - set all files to read and write by grader only
    `chmod 700 .ssh/` - set directory to read, write and executable by grader only
    * Run ssh as grader using port 2200
    `ssh grader@xx.xxx.xxx.xxx -p 2200` - this will prompt for a password.  Enter the password that was set during __ssh-keygen__.

### Prepare to deploy your project ###

9. Configure the local timezone to UTC.
`sudo dpkg-reconfigure tzdata`
Result:
_Current default time zone: 'Etc/UTC'
Local time is now:      Sat May  5 18:56:09 UTC 2018.
Universal Time is now:  Sat May  5 18:56:09 UTC 2018._

10. Install and configure Apache to serve a Python mod_wsgi application.
    * `sudo apt-get install apache2`
    * Verify by checking if homepage will display Ubuntu Apache home page - `http://34.216.227.195`
    * Install pre-requisite Apache components
`sudo apt-get install apache2 apache2-utils libexpat1 ssl-cert python`
    * Install mod_wsgi
    `sudo apt-get install libapache2-mod-wsgi`
    * Restart Apache service to make the mod_wsgi effective
    `sudo /etc/init.d/apache2 restart`

11. Install and configure PostgreSQL - `sudo apt-get install postgresql`
    * Do not allow remote connections
    Open `/etc/postgresql/9.5/main/pg_hba.conf` file.  Verify that its set as:
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
    * Create a new database user named __catalog__ that has limited permissions to your catalog application database.
    `sudo su - postgres`
    `psql`
    In postgres prompt, create a new role __catalog__ with Login permission  - `CREATE ROLE catalog WITH LOGIN;`
    Add __CREATEDB__ permission to __catalog__ role - `ALTER ROLE catalog CREATEDB;`
    Set password to catalog `\password catalog`
    * Create a new linux user __catalog__
    `sudo adduser catalog`
    * Give sudo access to linux user __catalog__
    `sudo nano /etc/sudoers` and add line `catalog ALL=(ALL:ALL) ALL`
    Verify that catalog has sudo access by logging in as catalog `sudo su - catalog` and running `sudo -l` as __catalog__


12. Install __git__
`sudo apt-get install git`

### Deploy the Item Catalog project ###

13. Clone and setup your Item Catalog project from the Github repository you created earlier in this Nanodegree program.
    * Create a directory called __catalog__ in /var/www
    ubuntu@ip-xxx-xx-x-xxx:/var/www$ `sudo mkdir catalog`
    * cd to /var/www/catalog and clone the catalog project
    ubuntu@ip-xxx-xx-x-xxx:/var/www/catalog$ `sudo git clone https://github.com/evelozud2017/catalog.git catalog` - This will clone the project into catalog folder under /var/www/catalog/catalog
    * Change permission of folder to have owner as ubuntu and group as ubuntu
    ubuntu@ip-xxx-xx-x-xxx:/var/www$ `sudo chown -R ubuntu:ubuntu catalog/`
14. Set it up in your server so that it functions correctly when visiting your server’s IP address in a browser. Make sure that your .git directory is not publicly accessible via a browser!
    * cd to /var/www/catalog/catalog directory
    * change the name of application.py file to \_\_init\_\_.py `mv application.py __init__.py`
    * Update \_\_init\_\_.py - change the line `app.run(host='0.0.0.0', port=8000)` into `app.run()`
    ##### Convert code from SQLLite to PostgreSQL #####
    * Update the following files in /var/www/catalog/catalog - itemcatalog_dbsetup.py, loadtestdata.py, \_\_init\_\_.py
    Change the line `engine = create_engine('sqlite:///itemcatalog.db')` into `engine = create_engine('postgresql://catalog:xxx@localhost/catalog')` where xxx is the catalog db password.
    ##### Generate clients_secret.json from Google #####
    * Create new project from [Google APIs](https://console.developers.google.com)
    * Create new OAuth credentials for the project
    * Add http://ec2-34-216-227-195.us-west-2.compute.amazonaws.com in the __Authorized JavaScript origins__
    * Add http://ec2-34-216-227-195.us-west-2.compute.amazonaws.com/login, http://ec2-34-216-227-195.us-west-2.compute.amazonaws.com/gconnect and http://ec2-34-216-227-195.us-west-2.compute.amazonaws.com/gdisconnect in __Authorized redirect URIs__

    ##### Update code with new client id and client secrets #####
    * Create __/var/www/catalog/catalog/client_secrets.json__ with values from client_secrets.json from Google API
    * Update data-clientid value of __/var/www/catalog/catalog/templates/login.html__ with the new client_id from Google API
    * Update the `client_secrets.json` declared to the following files from __/var/www/catalog/catalog/\_\_init\_\_.py__ into full path

    ```
    open(
        '/var/www/catalog/catalog/client_secrets.json', 'r'
    ).read())['web']['client_id']
    ...
    ...
    oauth_flow = flow_from_clientsecrets('/var/www/catalog/catalog/client_secrets.json', scope='')
    ```
    * Update the `client_secrets.json` declared to the following files from __/var/www/catalog/catalog/\_\_init\_\_.py__ into full path

    ##### Set up virtual env and install dependencies #####
    * Install pip `sudo apt-get install python-pip`
    * Install virtualenv `sudo apt-get install python-virtualenv`
    * `cd /var/www/catalog/catalog directory`
    * Create a new virtualenv `virtualenv tempenv`
    * Run the new environment
    ubuntu@ip-xxx-xx-x-xxx:/var/www/catalog/catalog$ `. tempenv/bin/activate`
    * Install the packages
    `pip install httplib2`
    `pip install requests`
    `pip install --upgrade oauth2client`
    `pip install sqlalchemy`
    `pip install flask`
    `sudo apt-get install libpq-dev`
    `pip install psycopg2`
    `pip install passlib`

    * Check that installation was successful - run `python __init__.py`
    __Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)__ - if this is displayed then its successful.
    * Stop application and deactivate virtual env - `deactivate`
    ##### Set up virtual env and enable a virtual host #####
    * Create a file called __catalog.conf__ in __/etc/apache2/sites-available/
    * Add the following lines:
    ```
    <VirtualHost *:80>
                ServerName XX.XXX.XXX.XXX
                ServerAdmin youremail@gmail.com
                WSGIScriptAlias / /var/www/catalog/catalog.wsgi
                <Directory /var/www/catalog/catalog/>
                        Order allow,deny
                        Allow from all
                        Options -Indexes
                </Directory>
                Alias /static /var/www/catalog/catalog/static
                <Directory /var/www/catalog/catalog/static/>
                        Order allow,deny
                        Allow from all
                        Options -Indexes
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```
    * Run `sudo a2ensite catalog` to enable the virtual host

    ##### Set up database schema and populate the database #####
    * While in /var/www/catalog/catalog/directory, activate the virtual env `. tempenv/bin/activate`
    * Run the python to setup db `python itemcatalog_dbsetup.py`
    * Load test data `python loadtestdata.py`
    * Deactivate virtualenv `deactivate`
    * Restart apache `sudo service apache2 restart`
    * Test the application thru browser - http://ec2-34-216-227-195.us-west-2.compute.amazonaws.com/

    ##### Disable the default Apache site #####
    * Run `sudo a2dissite 000-default.conf`
    * Restart apache `service apache2 reload`


# Sources
* DigitalOcean Tutorial - [UFW Essentials: Common Firewall Rules and Commands](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands)
* DigitalOcean Tutorial - [How To Install and Use PostgreSQL on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-18-04)
* DigitalOcean Tutorial - [How To Install the Apache Web Server on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-18-04)
* [Add new sudo user to EC2 Ubuntu](http://brianflove.com/2013/06/18/add-new-sudo-user-to-ec2-ubuntu/)
* Flask - [Install virtualenv](http://flask.pocoo.org/docs/1.0/installation/#install-install-virtualenv)
* Udacity Forum [Linux Server Configuration - SSH (Permission denied) and 404 Not Found](https://discussions.udacity.com/t/linux-server-configuration-ssh-permission-denied-and-404-not-found/439664)
* Udacity Forum [Item Catalog project dont work in the linux server configuration project](https://discussions.udacity.com/t/item-catalog-project-dont-work-in-the-linux-server-configuration-project/348571/9)
* Linux Man Pages [a2ensite](https://www.systutorials.com/docs/linux/man/8-a2ensite/)
* modwsgi [Quick Configuration Guide](https://modwsgi.readthedocs.io/en/develop/user-guides/quick-configuration-guide.html)
* libq-dev [Documentation](https://pypi.org/project/libpq-dev/)
* Psycopg [Documentation](http://initd.org/psycopg/docs/)
* Passlib [Documentation](https://passlib.readthedocs.io/en/stable/)
