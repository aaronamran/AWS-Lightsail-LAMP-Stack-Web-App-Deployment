# AWS Lightsail LAMP Stack Web Application Deployment
This is a writeup of a practical project that involves deploying a PHP web application with CRUD functionality onto an AWS Lightsail instance and configuring it to have a static public IP address with domain name and SSL certificate
1. [Create and Configure AWS Lightsail Instance](https://github.com/aaronamran/AWS-Lightsail-LAMP-Stack-Web-App-Deployment/tree/main#create-and-configure-aws-lightsail-instance)
2. [Allocate a Static IP Address](https://github.com/aaronamran/AWS-Lightsail-LAMP-Stack-Web-App-Deployment/tree/main#allocate-a-static-ip-address)
3. [Connect to Lightsail Instance via SSH](https://github.com/aaronamran/AWS-Lightsail-LAMP-Stack-Web-App-Deployment/tree/main#connect-to-lightsail-instance-via-ssh)
4. [Configure Apache Virtual Host for a Custom Domain](https://github.com/aaronamran/AWS-Lightsail-LAMP-Stack-Web-App-Deployment/tree/main#configure-apache-virtual-host-for-a-custom-domain)
5. [Secure Site with SSL](https://github.com/aaronamran/AWS-Lightsail-LAMP-Stack-Web-App-Deployment/tree/main#secure-site-with-ssl)
6. [Test Deployed Web Application](https://github.com/aaronamran/AWS-Lightsail-LAMP-Stack-Web-App-Deployment/tree/main#test-deployed-web-application)


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


## Secure Site with SSL
- However, when we visit the web app, we can see that it is not secure. This means that it runs on port 80 (HTTP), and should instead be running on port 443 (HTTPS) <br>
  ![image](https://github.com/user-attachments/assets/8e3c9592-4c12-4d82-a79b-ed7939609c73)

- Enter the browser-based SSH in Lightsail. Run the BNcert Tool. If there are any updates for the tool, follow the instructions
  ```
  sudo /opt/bitnami/bncert-tool
  ```
  ![image](https://github.com/user-attachments/assets/149874d2-d6e2-4cd7-99ac-ead6275e58e2)


- A series of prompts about redirections and other changes will appear
  ![image](https://github.com/user-attachments/assets/3f36b37c-72c4-4a9d-a768-b6435f20ac1c) <br>
  Note that if your hostname is registered without `www`, enabling www redirections to the hostname will cause the following error and failure of SSL certificate process. Since our registered hostname in No-IP is only `todolist.zapto.org`, registering `www.todolist.zapto.org` would also result in error as it does not exist <br>
  ![image](https://github.com/user-attachments/assets/c5fbb6ea-f935-4743-8f5f-31a2a37e4200)

  

## Test Deployed Web Application
- Now the LAMP stack web app is successfully deployed with SSL certificate
  ![image](https://github.com/user-attachments/assets/fa885040-d111-4a77-9e26-49cbac8a113d) <br>
  ![image](https://github.com/user-attachments/assets/16bf4833-232f-4213-8573-feefd4a729e9)

- Add a simple task for testing purposes
  ![image](https://github.com/user-attachments/assets/80bdbcdf-0e47-424c-a7a2-4c90e6443a12)

- When returning to the homepage, the added task is seen
  ![image](https://github.com/user-attachments/assets/b17c4b67-231a-43a0-a85b-2ecedca99af6)

- It is possible to edit the added task
  ![image](https://github.com/user-attachments/assets/e1801d4c-be64-454d-a293-bdbd90b3d021)

- Deleting the task is also part of the web application's functionality
  ![image](https://github.com/user-attachments/assets/b16edd60-7048-4f0b-94e8-8d3d1d983336)



- To prevent unwanted billing charges, return to Lightsail instances dashboard and stop and delete the specified instance <br> 
  ![image](https://github.com/user-attachments/assets/47ff587b-96d6-4c6c-857a-a2c07e8855fc)

