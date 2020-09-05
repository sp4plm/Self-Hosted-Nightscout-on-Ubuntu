# VPS-Hosted-Nightscout-on-Ubuntu 18

When you obtain your Ubuntu instance from VPS provider you receive root's login and root's password. From this moment follow the procedure.

## 1. Create a New User
Once you are logged in your Ubuntu instance as root, you are prepared to add the new user account that we will use to log in from now on.

This command creates a new user called “mainuser”, but you can replace it with a username that you like:

`# adduser mainuser`

Grant Root Privileges for the mainuser:

`# usermod -aG sudo mainuser`

Verify mainuser and its Privileges:

`# su mainuser`

```
$ grep '^sudo' /etc/group

sudo:x:27:ihc,mainuser
```

## 2. Prepare for installation

Update the Ubuntu instance:

`sudo apt-get update && sudo apt-get upgrade`

Check that Ubuntu instance has Python, npm, nodejs, git installed.
```
$ python3 --version
Python 3.6.9
```

Ubuntu 18.04 comes with pre-installed Python 3 as a default python interpreter. But we need python 2:

`$ sudo apt install python`

Then check installation:

```
$ python -V
Python 2.7.17
```

Then check git is installed:

```
$ git --version
git version 2.17.1
```


To check that nodejs is install run command:

 `$ dpkg --get-selections | grep node`

If you want to install another version of nodejs, remove the old one:

 `$ sudo apt purge nodejs`

### Install NODE.JS with node version manager

To install Node js on Ubuntu 18.04 with NVM we need C++ copiler and some other tools. To install them run command:

 `$ sudo apt install build-essential checkinstall`

We also need libssl:

 `$ sudo apt install libssl-dev`

To install NVM run:

`$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash`

Restart terminal or run:

`$ source /etc/profile`

Look throw Node js avalibale versions:

`$ nvm ls-remote`

Now install Node js. It obligatory to point a version of node js:

`$ nvm install 10.0`

A list of installed versions of node js can be viewed by:

`$ nvm list`

Tell the NVM what version of node js to use:

 `$ nvm use 10.0`
 
For further update to remove installed version it should be deactivated:

 `$ nvm deactivate 10.0`

And now can be removed:

 `$ nvm uninstall 10.0`

## Install Mongo DB

### Step 1 – Setup Apt Repository

First of all, import GPK key for the MongoDB apt repository on your system using the following command. This is required to test packages before installation

`$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 4B7C549A058F8B6B`

Lets add MongoDB APT repository url in /etc/apt/sources.list.d/mongodb.list.

`$ echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb.list`

### Step 2 – Install MongoDB on Ubuntu

After adding required APT repositories, use the following commands to install MongoDB on your systems. It will also install all dependent packages required for MongoDB.

`$ sudo apt update`

`$ sudo apt install mongodb-org`

If you want to install any specific version of MongoDB, define the version number as below. In order to migrate from Atlas I choose the same version of Mongo DB.

`$ sudo apt install mongodb-org=4.2.9 mongodb-org-server=4.2.9 mongodb-org-shell=4.2.9 mongodb-org-mongos=4.2.9 mongodb-org-tools=4.2.9`

### Step 3 – Manage MongoDB Service

After installation, MongoDB will start automatically. To start or stop MongoDB uses init script. Below are the example commands to do.

`$ sudo systemctl enable mongod`

`$ sudo systemctl start mongod`

Use the following commands to stop or restart MongoDB service.

`$ sudo systemctl stop mongod`

`$ sudo systemctl restart mongod`

### Step 5 – Verify MongoDB Installation

Finally, use the below command to check installed MongoDB version on your system.

```
$ mongod --version
  db version v4.2.8
  git version: 43d25964249164d76d5e04dd6cf38f6111e21f5f
  OpenSSL version: OpenSSL 1.1.1  11 Sep 2018
  allocator: tcmalloc
  modules: none
  build environment:
      distmod: ubuntu1804
      distarch: x86_64
      target_arch: x86_64
```      
Also, connect MongoDB using the command line and execute some test commands for checking proper working.

```
$ mongo
  > show dbs
    admin   0.000GB
    config  0.000GB
    local   0.000GB
  > use admin
   switched to db admin
  > quit()
```

### Step 6 – Configure MongoDB

Then create a new database and a new user:
```
$ mongo
> use Nightscout
> db.createUser({user: "mainuser", pwd: "YouPassword", roles:["readWrite"]})
> quit()
```

## Install CGM-Remote-Monitor (Nightscout)
Check you location:
```
$ pwd
> /home/mainuser
```

Download cgm-remote-monitor (nightscout) from github:

`$ git clone https://github.com/nightscout/cgm-remote-monitor.git`

Alternatively fork a copy of cgm-remote-monitor and clone your own copy.

`cd cgm-remote-monitor`

Install cgm-remote-monitor:

`$ git checkout dev`

`$ npm install` 

Now setup your cgm-remote-monitor environment and create start file:

`$ nano start.sh`

Paste those lines (a detailed description of the variables is here):

```
#!/bin/bash

# environment variables
export DISPLAY_UNITS="mg/dl"
export MONGO_CONNECTION="mongodb://mainuser:YouPassword@localhost:27017/Nightscout"
export BASE_URL="127.0.0.1:1337"
export API_SECRET="YOUR_API_SECRET_HERE"
export PUMP_FIELDS="reservoir battery status"
export DEVICESTATUS_ADVANCED=true
export ENABLE="careportal iob cob openaps pump bwg rawbg basal"
export TIME_FORMAT=24
export INSECURE_USE_HTTP=true

# start server
/home/mainuser/.nvm/versions/node/v10.16.3/bin/node server.js
```

We should make start.sh executable:

`$ chmod +100 +010 start.sh `

## Run Nightscout

You can now run Nightscout by entering:

`$ ./start.sh.`

## Create Reverse nginx proxy

Install nginx:

`$ sudo apt-get install nginx`

edit this file:

`$ sudo nano /etc/nginx/sites-available/default `

Delete the existing contents and replace with this:

```
server {
    listen 80;

    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:1337;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
Then restart the nginx service: 

`$ sudo service nginx restart`

## Create a service

I recommend using a systemd service which automatically starts Nightscout on system startup. To do so, create file:

`$ sudo nano /etc/systemd/system/nightscout.service`

and paste the following configuration:

```
[Unit]
Description=Nightscout Service      
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/mainuser/cgm-remote-monitor
ExecStart=/home/mainuser/cgm-remote-monitor/start.sh

[Install]
WantedBy=multi-user.target
```

Reload systemd:

`$ sudo systemctl daemon-reload`

Start and enable

`$ sudo systemctl enable nightscout.service`

`$ sudo systemctl start nightscout.service`

Finally check if the service is running:

`$ sudo systemctl status nightscout.service`

## Let's Encrypt SSL

install Let's Encrypt: 

`$ sudo apt-get install letsencrypt python-certbot-nginx`

### Obtain SSL certificate using webroot plugin

Stop ngnix service: 

`$ sudo service nginx stop`

Obtain letsencrypt certificate: 

`$ sudo letsencrypt certonly` 

```
How would you like to authenticate with the ACME CA?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: Nginx Web Server plugin - Alpha (nginx)
2: Spin up a temporary webserver (standalone)
3: Place files in webroot directory (webroot)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-3] then [enter] (press 'c' to cancel): 
> 2
```

Enter your domain name when prompted. This will create the certificates for your domain name. The certificates should now be available at "/etc/letsencrypt/live/your_domain_name"

Improve SSL security by generating a strong Diffie-Hellman group: 

`$ sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048`

Modify the nginx-file, filling in your domain name as in the example below:

`$ nano etc/nginx/sites-enabled/defaults`

```
server {
    listen 80;

    server_name yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:1337;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

        listen       443 ssl;

    server_name   yourdomain.com;
        root         /usr/share/nginx/html;

        ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_dhparam /etc/ssl/certs/dhparam.pem;
        ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM$
CDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AE$
HE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-EC$
84:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES$
RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-$
HA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aE$
S-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_stapling on;
        ssl_stapling_verify on;
        add_header Strict-Transport-Security max-age=15768000;

        location ~ /.well-known {
                allow all;
        }
}
```

Close the editor and check that your nginx configuration file is valid:

```
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If there are errors in your configuration file, the command's output will tell you exactly where in the file the error was found.

Restart nginx: 

`$ sudo service nginx restart`

You can test the quality of the SSL connection using: https://www.ssllabs.com/ssltest/analyze.html?d=your_domain_name Unfortunately only works with port 443

Arrange auto renewal of certificates. Add this line to the su crontab: 

`$ sudo crontab -e`

`30 2 * * 1 certbot renew >> /var/log/le-renew.log`

Hopefully your instance of Nightscout works fine!
