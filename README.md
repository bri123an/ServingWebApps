# Serving Your Web Applications

I believe that the process of deploying and running code is often trivialized. I view it as an extremely important part of the software development lifecycle. Running and deploying code is often a necessary part of developing code, so I would like to cover some topics that are useful and relevant to those wishing to set up a development server or even a production application.

*Warning: Some of what described below are not good security practices, definitely do not follow this guide to set up a production application*

*If at any point while following this guide you encounter a situation where you get an error message about permissions when attempting to run a command preface the command with sudo or re-run the last executed command with sudo by running
 ```sh
 sudo !!
 ```

---

## General Tasks For Running Web Applications Publicly

- Register a Domain Name
- Set up DNS Record/s
- Provision Server with a network interface that is accessible on the public internet
- Configure Server
- Configure Firewall
- Install/Configure Reverse Proxy
- Obtain SSL certificates and apply to Proxy configuration
- Install/Configure Database
- Install/Configure Application Runtime
- Move application code to server
- Run application

---

## Example Application - Node.js and MySQL on Ubuntu 20.04

<br>

I would recommend beginning by registering a domain and spinning up an instance of Ubuntu 20.04 on the cloud/vps provider of your choice. Make sure to look for free trial credit as many platforms offer credits to new users. I have personally used many different providers. I like using AWS for projects as it holds market dominance which makes experience gained highly transferable knowledge. I have used Azure and GCP as well. All of the major cloud providers have their strengths and weaknesses. Ultimately, this choice won't matter much, so choose what works best for you.

If you choose to use AWS I would recommend using Lightsail which is their simplified interface for provisioning servers. Choose the Memory and CPU configuration that makes the most sense for your use case. Generally Memory will be your limiting factor.

If you provision your server with AWS Lightsail, you should be able to see the public ip address for your server. You can create a DNS A Record with this public ip address.

<br>

---

<br>

## For the following, I'll assume you have a server running and a DNS record set up

<br>

On your Ubuntu server, run the following commands

```sh
sudo apt-get update

sudo apt-get upgrade -y

sudo apt install git-all

sudo apt install apache2

sudo apt install mysql-server

sudo snap install --classic node

sudo snap install --classic certbot

sudo apt install pm2
```
<br>

Check that node and npm are installed by running:

```sh
node -v
```

and

```sh
npm -v
```
<br>

Check that pm2 is installed by running:

```sh
pm2 -v
```

<br>

Check that MySQL is running by running:

```sh
sudo systemctl status mysql.service
```
<br>

Check that Apache is running by running:

```sh
sudo systemctl status apache2.service
```

<br>

After running the above commands you should have an Ubuntu 20.04 server with MySQL, Apache HTTP Server, and Node.js installed. We also have a process manager utility called PM2 installed which will help us run our Node process.


---

<br>

## Configuration

<br>

### **MySQL**

There are two places where we will need to do some configuration with respect to MySQL. We will need to make a change to one of the default config files to set the bind address. This will be necessary to connect to MySQL over TCP sockets from addresses other than localhost. The other configuration step we need to take in MySQL is to create a user and grant privileges to this user. In order to be able to use the npm module we will need to make sure that our user is identified with a mysql_native_password because the mysql node module is not compatible with the authentication used by the latest version of MySQL.

<br>
<br>

### Changing the default config file

<br>

You can use vi or nano, but I will reference nano as I think it is a more beginner-friendly text editor.

```
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

In this file look for the line that says bind-address and change it to the following.

```
bind-address = 0.0.0.0
```

This will let you connect to your MySQL instance over tcp sockets as opposed to Unix sockets and will let you connect from outside addresses. If you set the bind address to 127.0.0.1, MySQL will reject connections that don't originate from the host running it. This is simetimes what you want, but for a development server this is likely undesireable.

<br>

### Creating your database user

*The wildcard % is used so that you can use this user from many different hosts. This is probably what you want for developing and testing, but would not be a secure practice for a production application. I would also recommend restricting your users permissions on a production application instead of granting all permissions on everything.*

```sh
sudo mysql
```

Then from the mysql shell run the SQL below replacing 'your-username' and 'your-password' with the username and password you intent to use for your node.js application.

```SQL
CREATE USER 'your-username'@'%' 
IDENTIFIED WITH mysql_native_password
BY 'your-password';

GRANT ALL PRIVILEGES ON *.* TO 'your-username'@'%';
```

<br>

---

<br>


### **Node.js**

We will need to install any npm modules needed by your Node.js application. First navigate to the directory that contains your package.json file.
```sh
sudo npm install

pm2 server.js
```

Replace server.js with the name of the file containing your Node.js code.

<br>

### **Apache**


There will be several pieces of configuration that will need to be dealt with in Apache. We will need to enable some apache modules, set up our virtualhost configuration, and create and enable our ssl certificate that we will obtain with the certbot utility.

<br>

### Enabling Necessary Apache Modules

Check which Apache modules are installed

```sh
sudo apache2ctl -M

or

sudo apache2ctl -t -D DUMP_MODULES  
```

Enabling required modules

```sh
sudo a2enmod proxy

sudo a2enmod ssl
```

<br>

### Configure VirtualHost

<br>

Disable default VirtualHost
```
sudo a2dissite 000-default.conf
```

Navigate to the sites-available directory and create new VirtualHost config file.

```
cd /etc/apache2/sites-available

sudo nano node-app.conf
```

In this file you will provide the following configuration:

```
<VirtualHost *:80>

ServerName <name-that-you-set-up-dns-record-for> -Example node.site.com

ProxyPass / http://localhost:<replace-with-your-port>/ -Example http://localhost:3000/

</VirtualHost>
```
When you replace the above with your configuration do not include the angle brackets <>.

Enable your new configuration
```sh
sudo a2ensite node-app.conf

sudo systemctl reload apache2.service

sudo ufw allow 80
```

This should give you a site that is accessible over port 80 with the domain record you set up.

### Setting up SSL
If you would like to set up ssl there are a few extra steps.

Obtaining SSL certificate via openssl
```sh
sudo certbot certonly --apache
```
Follow the prompt to generate the certificates

<br>

You will need to make the following changes to your VirtualHost config file. Your configuration will look something like:

```
<IfModule mod_ssl.c>
<VirtualHost *:443>

ServerName <name-that-you-set-up-dns-record-for> -Example node.site.com

ProxyPass / http://localhost:<replace-with-your-port>/ -Example http://localhost:3000/

SSLCertificateFile /etc/letsencrypt/live/<replace-with-your-site-domain-name>/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/<replace-with-your-site-domain-name>/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
</IfModule>
```

Allow traffic on port 443
```
sudo ufw allow 443
```

Make sure to open the necessary ports in your cloud providers configuration as well. I use Amazon's Lightsail service to host some of my demo applications. To make the necessary changes for AWS Lightsail, go to the network tab and make new rules for ports 80 and 443 whichever you nbeed.

After this your application should be publicly accessible via the domain you set up with your DNS provider. If you set up an ssl certificate and modified your virtualhost configuration appropriately you should be able to access your application with https


