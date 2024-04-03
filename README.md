# Hosting flask application on google cloud using ubuntu, nginx, gunicorn, supervisor

* Create a project from google cloud
* create an instances with prefer os
* login through ssh from google cloud

#### Update and upgrade system
```bash
sudo apt update
sudo apt upgrade
```

#### Create a user and assign in sudo group to run admin commands
```bash
sudo adduser username		# create a new user
sudo adduser username sudo	# assign in sudo group

cat /etc/group			# check user added on sudo group
```

#### set ssh key based authentication for new user

##### from local linux system bash
```bash
ssh-keygen -t ed25519 -C 'ubuntu-server'		# create cryptographic hash key
# save this key /home/username/.ssh/id_flask_server	# id_flask_server, id_flask_server.pub

cat ~/.ssh/id_flask_server.pub 				# get the ssh public key
# copy the key and paste in cloud ubuntu machine 
# .ssh/authorized_keys directory

# this is not necessary
nano ~/.ssh/config					# add ssh configuration for easy access
-----------------------------------------|
Host flask-server			 |
    Hostname ip				 |		# copy external ip from google cloud
    User username			 |
    IdentityFile ~/.ssh/id_flask_server  |		# ssh keys path which we have created
-----------------------------------------|

# after puting the public key on cloud server
ssh flask-server					# login to remote server through ssh
```

##### from cloud server bash
```bash
su username				# switch to new user
	
mkdir ~/.ssh				# create a new .ssh directory
chmod 0700 .ssh				# update permission for .ssh directory

nano ~/.ssh/authorized_keys		# put the copied public key from local
chmod 0600 ~/.ssh/*			# update permission inside .ssh directory
exit					# exit from cloud machine
```

#### from local linux system bash
```bash
ssh flask-server			# login to our remote server

# if ssh working than we don't want root login and password login for security
sudo nano /etc/ssh/sshd_config
PermitRootLogin no			# disable root login
PasswordAuthentication no		# disable pass login

sudo systemctl restart sshd.service	# restart the service
```

#### clone flask application from github and configure to run
```bash
# clone the application
git clone https://github.com/mahfuz-prog/Flask.git	# clone in ~/ directory

# install virtual environment on system
sudo apt install python3-venv

# create virtual env on Flask package
cd Flask
python3 -m venv .env

source .env/bin/activate 				# activate virtual env
pip install -r requirements.txt				# install dependencies for app
rm ~/Flask/requirements.txt				# not to keep sensitive info on server
```

#### Test application on server
##### [update google cloud firewall](https://console.cloud.google.com/net-security/firewall-manager/firewall-policies/)
##### add new firewall rule to allow 5000 port for test
###### name
* allow-5000
###### Targets 
* all instancex in the network
* Source IPv4 ranges = 0.0.0.0/0
###### Protocols
* select Specified protocols and ports
* select TCP = 5000

```bash
export FLASK_APP=run.py		# temp variable
flask run --host=0.0.0.0	# access development server from outside

go to ipaddress:5000 to see the application is running
```
##### delete allow-5000 from google cloud. we don't need 5000 port further. the same way allow 80 port `TCP=80` from google cloud for run app in production

#### set up firewall on our server
##### the firewall rule we set on google cloud is applicable for all instances of a project and this configuration is applicable for only our remote machine
```bash
sudo apt install ufw			# install ufw
sudo ufw status				# see the status

sudo ufw default allow outgoing		# allow outgoing traffic from server
sudo ufw default deny incoming		# deny all incoming traffic
sudo ufw allow ssh			# allow ssh port for ssh login
sudo ufw allow http/tcp			# allow port 80 for production
sudo ufw enable				# enable firewall
sudo ufw disable			# disable ufw
```

#### set up nginx reverse proxy
```bash
sudo apt install nginx				# install nginx

# remove default nginx file
sudo rm /etc/nginx/sites-enabled/default

# create new nginx file
sudo nano /etc/nginx/sites-enabled/flaskapp

server {
	listen 80;
	server_name ip;
	
	location /static {
		alias /home/username/Flask/flaskapp/static;
	}
	
	location / {
		proxy_pass http://localhost:8000;
		include /etc/nginx/proxy_params;
		proxy_redirect off;
	}
}

sudo systemctl restart nginx			# restart nginx
```
##### visit the ip address it should gives nginx error because it still doesn't know how to process python code. Now we need gunicorn

#### gunicorn setup 
```bash
pip install gunicorn			# install gunicorn in virtual env

nproc --all				# number of cores
worker = (2 * number of cores) + 1

# run has our flask application variable which is app
# run gunicorn
gunicorn -w 5 run:app			# check the app running
```

#### Supervisor process manager
##### helps to keep runing our app and autostart if server crash
```bash
sudo apt install supervisor		# install supervisor

# create a config file
sudo nano /etc/supervisor/conf.d/flaskapp.conf

[program:flaskapp]
directory=/home/username/Flask/
command=/home/username/Flask/.env/bin/gunicorn -w 5 flaskapp:app
user=username
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
stderr_logfile=/var/log/flaskapp/flaskapp.err.log
stdout_logfile=/var/log/flaskapp/flaskapp.out.log
```

##### create those file and directory defined on supervisor config
```bash
sudo mkdir -p /var/log/flaskapp/
sudo touch /var/log/flaskapp/flaskapp.err.log
sudo touch /var/log/flaskapp/flaskapp.out.log
```

##### before start supervisor we need one more setting to go
##### [hostname resolution](https://www.debian.org/doc/manuals/debian-reference/ch05.en.html#_the_hostname_resolution)
```bash
hostname 			# check the hostname
hostnamectl set-hostname name	# set a hostname

# edit hosts file and put ip address with hostname below localhost
sudo nano /etc/hosts

127.0.0.1 localhost
yourip     hostname
```

#### Run the flask app on production
```bash
sudo supervisorctl reload		# start running the application
sudo supervisorctl status		# check the status of supervisor
```

## Quick recap
* `cat /etc/group` check user in sudo group.
* `ls -la ~` .ssh permission should `700(drwx------)` for .ssh directory of our remote server.
* `ls -la ~/.ssh` authorized_keys which contain ssh public key should have `600(-rw-------)` permission.
* `cat /etc/ssh/sshd_config` PermitRootLogin and PasswordAuthentication should no
* `ls /etc/nginx/sites-enabled/` there should be only one file which contain nginx configuration
* `cat /etc/nginx/sites-enabled/flaskapp` check nginx config
* `cat /etc/supervisor/conf.d/flaskapp.conf` supervisor configuration
* `ls /var/log/flaskapp/` supervisor logfile
* 'cat /var/log/flaskapp/flaskapp.err.log' check the log file of supervisor
* `cat /etc/hosts` there should be external ip address with hostname
* `ssh, http/tcp` should allow on [google cloud firewall](https://console.cloud.google.com/net-security/firewall-manager/firewall-policies/)
* `sudo ufw status`
```bash
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
```