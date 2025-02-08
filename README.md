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


- 


## Connect to Lightsail Instance via SSH
- The default password to sign in to your database in LAMP is stored on your instance. Retrieve it by connecting to your instance using the browser-based SSH terminal in the Lightsail console and running a special command

- On the Instances section of the Lightsail home page, choose the SSH quick-connect (terminal) icon for your LAMP instance <br>
  ![image](https://github.com/user-attachments/assets/92bd16bc-3924-4747-b6cc-b5b415c0d61b)

- After the browser-based SSH client window opens, enter the following command to retrieve the default application password:
  ```
  cat bitnami_application_password
  ```
  If you're in a directory other than the user home directory, then enter `cat $HOME/bitnami_application_password` <br>
  ![image](https://github.com/user-attachments/assets/b217dff2-1e31-4308-8836-b251832749d4)

- 


## Configure Apache Virtual Host for a Custom Domain



## Secure Site with SSL



## Test Deployed Web Application
