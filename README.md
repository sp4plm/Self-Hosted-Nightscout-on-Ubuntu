# Self-Hosted-Nightscout-on-Ubuntu

Update the Ubuntu instance:
`sudo apt-get update && sudo apt-get upgrade`

Update node:
```
sudo npm cache clean -f
sudo npm install -g n
sudo n stable
```
Create a New User
Once you are logged in as root, we’re prepared to add the new user account that we will use to log in from now on.

This example creates a new user called “sammy”, but you should replace it with a username that you like:

adduser sammy
Now, we have a new user account with regular account privileges. However, we may sometimes need to do administrative tasks.
To add these privileges to our new user, we need to add the new user to the “sudo” group.
`# usermod -aG sudo sammy`
