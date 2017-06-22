# Linux-Server-Configuration

## Project Overview
Prepare the server to host your web applications.

## Keypoints of the project
  - Deploying a web application to a publicly accessible server.
  - Properly securing application ensures, application remains stable and that user’s data is safe.
  
###### IP Address:
    13.59.45.138
###### SSH Port:
    2200
###### Link to Item Catalog Website:
   [ItemCatalogApp](http://ec2-13-59-45-138.us-east-2.compute.amazonaws.com/restaurant/)
   
## Steps Followed

1. Create new development environment instance.
   - Create new instance.
   - Download the private key.
  
2. Launch VM and ssh into the server.
   - Move private key into a folder.
   - Open terminal in that folder and follow the steps below.
   - Change the file rights to read only. Your key must not be publicly viewable for SSH to work.
   
          chmod 400 ./restaurant.pem
   - ssh into instance.
   
          ssh -i ./restaurant.pem root@PUBLIC_IP_ADDRESS
          
3. Create a new user account named "grader".
   - Add new user
          
         sudo adduser grader
   - Give grader the permission to sudo
   
        * Create a new file
          
              sudo nano /etc/sudoers.d/grader
        * Add following text in the above file.
            
              grader ALL=(ALL:ALL)ALL
   - Edit hosts file
   
              sudo nano /etc/hosts
   - Add host to hosts file
   
              127.0.1.1 ip-XX-XX-XX-XX

4. Setup ssh keys for user grader.
   - To sort out the virtual login issue. 
       * Move to sshd_config file 
       
             sudo nano /etc/ssh/sshd_config
       * In the above file make the following changes
       
             - set password authentication to "yes"
            
       * To restart the service run the following command.  
          
             sudo service ssh restart
                  
   - On the local system, Go to the directory where you want to save the Key, and run the following command.
       
           ssh-keygen -t rsa
   
   - Run the following command to install the generated public key on the server.
   
          ssh-copy-id -i grader.pub grader@PUBLIC_IP_ADDRESS
          
   - Now you are able to log into the remote VM (log into grader) through ssh with the following command.
 
          ssh -i grader -p 2200 grader@XX.XX.XX.XX
          
5. Enforce key-based authentication | Change the SSH port from 22 to 2200 | Disable login for root user.

   Run sudo nano /etc/ssh/sshd_config .

   Find the PasswordAuthentication line and edit it to no.

   Find the Port line and edit it to 2200.

   Find the PermitRootLogin line and edit it to no, Then save the file.

   Run sudo service ssh restart to restart the service.

          
6. Change timezone to UTC.

        sudo timedatectl set-timezone UTC
        
7. Updating all packages on the server

        sudo apt-get update
        sudo apt-get upgrade
        
8. Configure Firewall

      - Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), 
        HTTP (port 80), and NTP (port 123)  
        
            sudo ufw default deny incoming
            sudo ufw default allow outgoing
            sudo ufw allow 2200/tcp
            sudo ufw allow www
            sudo ufw allow ntp
            sudo ufw enable
            
9. Configure cron scripts to automatically manage package updates.
       
      - Install unattended-upgrades if not already installed:
            
             sudo apt-get install unattended-upgrades
      - To enable it, do:
      
             sudo dpkg-reconfigure --priority=low unattended-upgrades
             
10. Install and Configure Apache2 and mod-wsgi and Git.

        sudo apt-get install apache2 libapache2-mod-wsgi git
        
11. Install and configure PostgreSQL.

     - Installing PostgreSQL Python dependencies:
         
           sudo apt-get install libpq-dev python-dev
     - Installing PostgreSQL:
       
           sudo apt-get install postgresql postgresql-contrib
     - Login as postgres User, and get into PostgreSQL shell:
     
           sudo su - postgres
           psql
     
     - Create a new User named catalog: 
     
            CREATE USER catalog WITH PASSWORD 'sillypassword';

     - Create a new DB named catalog: 
            
            CREATE DATABASE catalog WITH OWNER catalog;

     - Connect to the database catalog : 
          
            \c catalog

     - Revoke all rights: 
       
            REVOKE ALL ON SCHEMA public FROM public;

     - Give access to only catalog role:
         
            GRANT ALL ON SCHEMA public TO catalog;

     - Log out from PostgreSQL: 
     
            \q
             
     - Then return to the grader user: 
        
            exit

     - Inside the Flask application, the database connection is now performed with:
     
           engine = create_engine('postgresql://catalog:sillypassword@localhost/catalog')
           
     - Inside the flask application, change the path of client_ID to:
     
           /var/www/catalog/client_secrets.json
           
12. Install Flask and other dependencies

        sudo apt-get install python-pip
        sudo pip install Flask
        sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils requests

        
13. Clone the Item Catalog app from Github

    - Make a catalog named directory in /var/www

          sudo mkdir /var/www/catalog

    - Change the owner of the directory catalog to grader.

          sudo chown -R grader:grader /var/www/catalog

    - Clone the Item Catalog to the catalog directory:

          git clone https://github.com/ashishu1396/Item-Catalog.git catalog

    - Make a catalog.wsgi file to serve the application over the mod_wsgi.:

          touch catalog.wsgi && nano catalog.wsgi
          
    - Add the following contents to the file:
     
          import sys
          import logging
          logging.basicConfig(stream=sys.stderr)
          sys.path.insert(0, "/var/www/catalog/")

          from project import app as application

    - Inside project.py database connection is now performed with:

          engine = create_engine('postgresql://catalog:sillypassword@localhost/catalog')

14.  Edit the default Virtual File with following content:

    - Open the file below:
     
          sudo nano /etc/apache2/sites-available/000-default.conf

    - Make the following changes in the above file:
    
          <VirtualHost *:80>
            ServerName XX.XX.XX.XX
            ServerAdmin Ashish
            WSGIScriptAlias / /var/www/catalog/catalog.wsgi
           <Directory /var/www/catalog/>
                Order allow,deny
                Allow from all
           </Directory>
           Alias /static /var/www/catalog/catalog/static
           <Directory /var/www/catalog/static/>
                Order allow,deny
                Allow from all
           </Directory>
           ErrorLog ${APACHE_LOG_DIR}/error.log
           LogLevel warn
           CustomLog ${APACHE_LOG_DIR}/access.log combined
          </VirtualHost>

    - Restart Apache to launch the app
        
          sudo service apache2 restart
