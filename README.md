# AWS Lightsail LAMP Stack Web Application Deployment
This is a writeup of a practical project that involves the following main concepts in order:
1. [Create and Configure AWS Lightsail Instance]()
2. [Allocate a Static IP Address]()
3. [Connect to Lightsail Instance via SSH]()
4. [Configure Apache Virtual Host for a Custom Domain]()
5. [Secure Site with SSL]()
6. [Test Deployed Web Application]()


## Create and Configure AWS Lightsail Instance
- In AWS, create a free tier account and complete the identity verification and billing information
- Log into the AWS Lightsail Console and click on `Create Instance` button <br>
  ![lightsail](https://github.com/user-attachments/assets/18dd304e-9a5d-4def-a129-9ce9d4b3d885)

- Choose the preferred instance location, platform, and blueprint. Since we will be deploying a LAMP stack web app, choose `Apps + OS` for the blueprint and select `LAMP (PHP 8)`
  ![image](https://github.com/user-attachments/assets/57158038-66c4-43e0-b747-0f65c6bf8527)

- A launch script is used to install the application code. Launch scripts run the first time an instance boots up, and are used to perform initial configurations on an instance. For a simple use case, the script below can be pasted into the text window
  ```
  # remove default website
  #-----------------------
  cd /opt/bitnami/apache2/htdocs 
  rm -rf *
  
  # clone github repo
  #------------------
  git clone -b loft https://github.com/mikegcoleman/todo-php .
  
  # set write permissions on the settings file
  #-----------------------------------
  sudo chown bitnami:daemon connectvalues.php
  chmod 666 connectvalues.php
  
  # inject database password into configuration file
  #-------------------------------------------------
  sed -i.bak "s/<password>/$(cat /home/bitnami/bitnami_application_password)/;" /opt/bitnami/apache2/htdocs/connectvalues.php
  
  # create database
  #----------------
  cat /home/bitnami/htdocs/data/init.sql | /opt/bitnami/mariadb/bin/mysql -u root -p$(cat /home/bitnami/bitnami_application_password)
  ```
  The script performs the following actions:
  - Removes the default Apache website
  - Clones the application code from GitHub into the htdocs directory
  - Ensures the configuration file is writeable
  - Uses sed to read the local database password from a file on the disk and insert it into the configuration file
  - Runs an SQL script to set up the application’s database
  <br>
  ![image](https://github.com/user-attachments/assets/ea835aa4-dee4-4303-a624-7a562f6c4b55) <br>

- However, for the sake of practical learning, we will be using our own custom PHP Web Application. That means we can modify the launch script as the following:
    ```
    #!/bin/bash
    # This launch script automates deployment of a PHP web app on an AWS Lightsail Bitnami LAMP stack.
    # Paste this script into the "User data" field when creating your Lightsail instance.
    
    # -----------------------------
    # (Optional) Update the System
    # -----------------------------
    apt-get update -y
    apt-get upgrade -y

    # -----------------------------------
    # Install Subversion (SVN) if not already installed, and is intended for use
    # -----------------------------------
    # apt-get install -y subversion
    
    # -----------------------------------
    # Remove the Default Website Content
    # -----------------------------------
    cd /opt/bitnami/apache2/htdocs || { echo "Directory not found!"; exit 1; }
    rm -rf *
    
    # -----------------------------------------
    # Deploy the Application Code from a Repo
    # -----------------------------------------
    # If you want to deploy your app using Git, uncomment the next line
    # and update <branch> and <repository_url> as needed.
    git clone -b main https://github.com/aaronamran/sample-php-crud .
    
    # ---------------------------------------------------
    # Set Write Permissions on the Application Settings File
    # ---------------------------------------------------
    # The file "connectvalues.php" (or your equivalent configuration file)
    # needs to be writable so that the database password can be injected.
    chown bitnami:daemon connectvalues.php
    chmod 666 connectvalues.php
    
    # ----------------------------------------------------
    # Inject the Database Password into the Configuration File
    # ----------------------------------------------------
    # This replaces the placeholder <password> in connectvalues.php with the
    # actual password stored in /home/bitnami/bitnami_application_password.
    # Make sure your configuration file contains the literal placeholder <password>.
    sed -i.bak "s/<password>/$(cat /home/bitnami/bitnami_application_password)/" /opt/bitnami/apache2/htdocs/connectvalues.php
    
    # ------------------------------
    # Create and Initialize the Database
    # ------------------------------
    cat /home/bitnami/htdocs/data/init.sql | /opt/bitnami/mariadb/bin/mysql -u root -p$(cat /home/bitnami/bitnami_application_password)
    
    # ------------------------------
    # (Optional) Restart Apache to Ensure Changes Take Effect
    # ------------------------------
    /opt/bitnami/ctlscript.sh restart apache
    ```
- Since this project is for learning purposes, we will go with the instance plan which has the lowest cost and is included in the free tier option
  ![image](https://github.com/user-attachments/assets/5d492538-1a8d-4313-b761-eb866d590989)

- Names and tags can be customised for the configured instance
  ![image](https://github.com/user-attachments/assets/66106d43-4e77-476a-90a3-41699688d33b)

- After clicking on `Create instance` button, the instance will start, and the following screen outcome can be seen
  ![image](https://github.com/user-attachments/assets/9dedf5ac-558f-4f59-9807-6ccba43f9f81)



## Allocate a Static IP Address
- The default public IP for your LAMP instance changes if you stop and start the instance. A static IP address, attached to an instance, stays the same even if you stop and start your instance
- To allocate a static IP address, open the LAMP instance. Choose the `Networking` tab and then `Attach static IP`
  ![image](https://github.com/user-attachments/assets/1ef9b08a-b1bb-4a9f-8e28-f54839036094) <br>
  ![image](https://github.com/user-attachments/assets/fb656ae6-86db-4e4d-ae93-0b9b70cc6e08)


- Name the static IP and click `Create and attach`
  ![image](https://github.com/user-attachments/assets/99a78e66-5284-4bbb-b5c2-5804d047b021)



## Connect to Lightsail Instance via SSH
- The default password to sign in to your database in LAMP is stored on your instance. Retrieve it by connecting to your instance using the browser-based SSH terminal in the Lightsail console and running a special command

- On the Instances section of the Lightsail home page, choose the SSH quick-connect (terminal) icon for your LAMP instance <br>
  ![image](https://github.com/user-attachments/assets/be90e9c6-7792-4b83-852e-473dc20e9e66)

- After the browser-based SSH client window opens, enter the following command to retrieve the default application password:
  ```
  cat bitnami_application_password
  ```
  If you're in a directory other than the user home directory, then enter `cat $HOME/bitnami_application_password` <br>
  ![image](https://github.com/user-attachments/assets/a0a08a4f-b884-4884-8943-7bf20d93c6f0)


## Configure Apache Virtual Host for a Custom Domain
- When we visit the public IP address, we can see that it has no hostname, only the IP address itself in the browser's URL bar
  ![image](https://github.com/user-attachments/assets/026c840c-ccf9-4f73-82e2-8046d9604859)

- For this project, we will go with a free hostname uisng [No-IP](https://my.noip.com/)
- After signing up for No-IP, navigate to the account dashboard and add a host
  ![image](https://github.com/user-attachments/assets/fc88d47b-c9be-4a75-ad15-f94863fd2089)

- Give a domain name for the web app, and enter the deployed LAMP web app public IP address
  ![image](https://github.com/user-attachments/assets/d172e420-7878-4fab-90a2-619471df2610)

- After saving the settings, create a DDNS key for the hostname. For reference, visit this [link](https://www.noip.com/support/knowledgebase/how-to-setup-and-use-a-ddns-key)
  ![image](https://github.com/user-attachments/assets/cb4719ec-f4e7-4924-b9bd-4be6ba2c3316)

- Clicking continue will save the settings and activate the web app's hostname. Now when we visit the web app, we can see the new name appearing
  ![image](https://github.com/user-attachments/assets/ec2f79a1-f3fc-4e76-8c00-041a24f36c17)

- Even though the domain (todolist.zapto.org) is pointing to the Lightsail instance’s static IP, Apache doesn't necessarily need a custom virtual host to serve the site. It simply falls back to the default virtual host if it doesn't find one that explicitly matches the domain in the request
- Connect to Lightsail's browser-based SSH. Then navigate to the Apache Configuration Diretory
  ```
  cd /opt/bitnami/apache2/conf/vhosts
  ```
  If the `vhosts` directory does not exist, it can be created
  ```
  sudo mkdir -p /opt/bitnami/apache2/conf/vhosts
  ```
- Create a file named `todolist-vhost.conf`
  ```
  sudo nano /opt/bitnami/apache2/conf/vhosts/todolist-vhost.conf
  ```
- Insert the following configuration
  ```
  <VirtualHost *:80>
    ServerName todolist.zapto.org
    ServerAlias www.todolist.zapto.org
    DocumentRoot "/opt/bitnami/apache2/htdocs"

    <Directory "/opt/bitnami/apache2/htdocs">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog "/opt/bitnami/apache2/logs/todolist_error.log"
    CustomLog "/opt/bitnami/apache2/logs/todolist_access.log" combined
  </VirtualHost>
  ```
- Depending on Bitnami configuration, it is good to ensure that custom vhost files are included in the main configuration. Open `/opt/bitnami/apache2/conf/bitnami/bitnami-apps-vhosts.conf`
  ```
  sudo nano /opt/bitnami/apache2/conf/bitnami/bitnami-apps-vhosts.conf
  ```
  Add the following line if it's not already there
  ```
  Include "/opt/bitnami/apache2/conf/vhosts/todolist-vhost.conf"
  ```
  To apply changes, restart Apache with the Bitnami control script
  ```
  sudo /opt/bitnami/ctlscript.sh restart apache
  ```


## Secure Site with SSL
- However, when we visit the web app, we can see that it is not secure. This means that it runs on port 80 (HTTP), and should instead be running on port 443 (HTTPS) <br>
  ![image](https://github.com/user-attachments/assets/8e3c9592-4c12-4d82-a79b-ed7939609c73)

- Enter the browser-based SSH in Lightsail. Navigate to `/opt/bitnami`
  ![image](https://github.com/user-attachments/assets/4fdf606a-2a8c-400f-bc55-d8217b5325db)

- Run the BNcert Tool
  ```
  sudo ./bncert-tool
  ```
  It will inform that an updated version is available for download. Then run
  ```
  sudo /opt/bitnami/bncert-tool
  ```
  ![image](https://github.com/user-attachments/assets/b6919a2f-0948-40fb-a173-e471856c592b)

- A series of prompts about redirections and other changes will appear
  ![image](https://github.com/user-attachments/assets/75aa4815-3f73-49c8-8361-bd0efbc883ae) <br>
  Note that if your hostname is registered without `www`, enabling www redirections to the hostname will cause the following error and failure of SSL certificate process <br>
  ![image](https://github.com/user-attachments/assets/824ef41b-ce7a-4691-9c31-f4024222e6a3)

- Navigate to `/opt/bitnami/apache2/conf/vhosts` directory and create a `todolist-vhost.conf` file. Paste the following configuration:
  ```
  # HTTP VirtualHost: Redirect all requests to HTTPS
  <VirtualHost *:80>
      DocumentRoot "/opt/bitnami/apache2/htdocs"
      ServerName todolist.zapto.org
      ServerAlias www.todolist.zapto.org
  
      # BEGIN: Enable HTTP to HTTPS redirection
      RewriteEngine On
      RewriteRule ^/(.*) https://todolist.zapto.org/$1 [R,L]
      # END: Enable HTTP to HTTPS redirection
  </VirtualHost>
  
  # HTTPS VirtualHost: Serve the site securely and redirect www to non-www
  <VirtualHost _default_:443>
      DocumentRoot "/opt/bitnami/apache2/htdocs"
      ServerName todolist.zapto.org
      ServerAlias www.todolist.zapto.org
  
      # BEGIN: Enable www to non-www redirection
      RewriteEngine On
      RewriteCond %{HTTP_HOST} !^todolist\.zapto\.org$ [NC]
      RewriteRule ^(.*)$ https://todolist.zapto.org$1 [R=permanent,L]
      # END: Enable www to non-www redirection
  
      <Directory "/opt/bitnami/apache2/htdocs">
          Options -Indexes +FollowSymLinks -MultiViews
          AllowOverride All
          Require all granted
      </Directory>
  
      SSLEngine on
      SSLCertificateFile "/opt/bitnami/apache2/conf/todolist.zapto.org.cert"
      SSLCertificateKeyFile "/opt/bitnami/apache2/conf/todolist.zapto.org.key"
  </VirtualHost>
  ```

- Then navigate to `/opt/bitnami/apache2/conf/bitnami/bitnami.conf` and add an include directive at the bottom
  ```
  # Custom Virtual Hosts
  Include "/opt/bitnami/apache2/conf/vhosts/todolist-vhost.conf"
  ```

  

## Test Deployed Web Application
