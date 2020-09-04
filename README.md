# Self-Hosted-Nightscout-on-Ubuntu

Update the Ubuntu instance:
`sudo apt-get update && sudo apt-get upgrade`

Update node:
```
sudo npm cache clean -f
sudo npm install -g n
sudo n stable
```
## Create a New User with root privilege
Once you are logged in as root, we’re prepared to add the new user account that we will use to log in from now on.

This example creates a new user called “sammy”, but you should replace it with a username that you like:

`# adduser sammy`

Now, we have a new user account with regular account privileges. However, we may sometimes need to do administrative tasks.
To add these privileges to our new user, we need to add the new user to the “sudo” group.

`# usermod -aG sudo sammy`


## Install CGM-Remote-Monitor (Nightscout)
Install Node.js and npm  `sudo apt-get install nodejs npm`

Download cgm-remote-monitor (nightscout) from github:
`git clone https://github.com/nightscout/cgm-remote-monitor.git`
Alternatively fork a copy of cgm-remote-monitor and clone your own copy.

`cd cgm-remote-monitor`

Install cgm-remote-monitor:
`git checkout dev`
`npm install` 

setup your cgm-remote-monitor environment as you normally would, for example creating a file my.env :
```
MONGO_CONNECTION=MONGOCONNECTIONSTRING
DISPLAY_UNITS=mmol
BASE_URL=NIGHTSCOUT_SITE_URL
DEVICESTATUS_ADVANCED="true"
mongo_collection="mogocollection_name"
API_SECRET=AVeryLongString
ENABLE=careportal%20openaps%20iob%20bwp%20cage%20basal%20pump%20bridge
BRIDGE_SERVER=EU
BRIDGE_USER_NAME=USERNAME
BRIDGE_PASSWORD=PASSWORD
```

## Install pm2 to monitor nightscout processs
`sudo npm install pm2 -g`

Start cgm-remote-monitor with pm2:
`env $(cat my.env)  PORT=1337 pm2 start server.js`

Make pm2 start cgm-remote-monitor on startup
`pm2 startup ubuntu` - this will give you a command you need to run as superuser to allow pm2 to start the app on reboot 
The command will be something like:
`sudo su -c "env PATH=$PATH:/usr/bin pm2 startup ubuntu -u username --hp /home/username"`
And then:
`pm2 save`

## Create Reverse nginx proxy

Install nginx:

`sudo apt-get install nginx`

edit this file:

`sudo vi /etc/nginx/sites-available/default `
Delete the existing contents and replace with this:
I'm assuming the proxy is on the same host as nightscout and the `proxy_pass http://127.0.0.1:1337` line - `1337` is replaced with the port that nightscout is using

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

Then restart the nginx service 
`sudo service nginx restart`
