<!---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
--->

# LNDg

[LNDg](https://github.com/cryptosharks131/lndg) is a lite GUI web interface to help you manually manage your node and automate operations such as rebalancing, fee adjustments and channel node opening.

{% hint style="danger" %}
Difficultry: Hard
{% endhint %}
<!--- need to upload the image to the gitbook assets folder
<figure><img src="../../.gitbook/assets/lndg.png" alt=""><figcaption></figcaption></figure>
--->
## Requirements

* [Bitcoin Core](../../bitcoin/bitcoin/bitcoin-client.md)
* [LND](../../lightning/lightning-client.md) 
* [PostgreSQL](../system/postgresql.md)
* Others
  * [venv](https://docs.python.org/3/library/venv.html)
  * [uWSGI](https://uwsgi-docs.readthedocs.io/)

## Preparations

### Firewall

* Configure the firewall to allow incoming HTTP requests from anywhere to the web server

```bash
sudo ufw allow 8889/tcp comment 'allow LNDg SSL from anywhere'
```

### PostgreSQL

* With user `admin`, check if you already have PostgreSQL installed

```bash
psql -V
```

**Example** of expected output:

```
psql (PostgreSQL) 17.0 (Ubuntu 17.0-1.pgdg22.04+1)
```

{% hint style="info" %}
If you obtain "**command not found**" outputs, you need to follow the [PostgreSQL bonus guide installation process](../bonus-guides/system/postgresql.md#installation) before continuing with this guide
{% endhint %}


#### Create PostgreSQL database for LNDg

* With user `admin`, create a new database for LNDg as the `postgres` user and assign the user `admin` as the owner

```bash
sudo -u postgres createdb -O admin lndg
```

#### Prepare for PostgreSQL integration

* Get required packages to build [psycopg2](https://www.psycopg.org/docs/) - used by LNDg to connect with PostgreSQL

```bash
sudo apt install gcc python3-dev libpq-dev
```

### Python virtual environment

[venv](https://docs.python.org/3/library/venv.html) is a module that supports creating lightweight "virtual environments". We will use ***venv*** to islolate the process of building and running the tools associated with LNDg so that we can avoid conflicts with our system's Python installation and other Python projects. 

* With user `admin`, install the python3-venv package

```bash
sudo apt install python3-venv
```

## Installation

### Create the lndg user & group

We do not want to run LNDg code alongside `bitcoind` and `lnd` because of security reasons. For that, we will create a separate user and run the code as the new user. We will install LNDg in the home directory since it doesn't need much space.

* Create the `lndg` user and group
  
```bash
sudo adduser --disabled-password --gecos "" lndg
```

* Add `lndg` user to the lnd and www-data groups

```bash
sudo usermod -a -G lnd,www-data lndg
```

* Add `www-data` user to the lndg group

```bash
sudo adduser www-data lndg
```

* Allow `lndg` group execute permissions on home directory

```bash
sudo chmod 710 /home/lndg/
```

* Change to the `lndg` user

```bash
sudo su - lndg
```

* Create a symbolic link to the lnd data directory

```bash
ln -s /data/lnd /home/lndg/.lnd
```

* Confirm symbolic link has been created

```bash
ls -la .lnd
```

Expected output: 
<!--- need to fix for gitbook --->

<pre><code>lrwxrwxrwx 1 lndg lndg 9 Nov 10 01:03 <a data-footnote-ref href="#user-content-fn-10">.lnd -> /data/lnd</a></code></pre>

<!--- OLD STYLE

* Check the version number of the latest LNDg release (you can also confirm with the [release page](https://github.com/cryptosharks131/lndg/releases))

```bash
LATEST_RELEASE=$(wget -qO- https://api.github.com/repos/cryptosharks131/lndg/releases/latest | grep -oP '"tag_name":\s*"\K([^"]+)') && echo $LATEST_RELEASE
```  

**Example** of expected output:

```
v1.9.1
```
--->

* Set a temporary version environment variable to the installation

```bash
VERSION=1.9.1
```
<!---

As of v1.9.1, the developer has not renewed the GPG key. The following steps will be updated when the developer renews the key.

* Import the GPG key of the developer

```bash
curl https://github.com/cryptosharks131.gpg | gpg --import
```
--->
* Import the GitHub web flow GPG public key

```bash
gpg --keyserver keyserver.ubuntu.com --recv-keys 968479A1AFF927E37D1A566BB5690EEEBB952194
```
 
* Download the source code directly from GitHub, select the latest associated release branch, and change to the `lndg` folder

```bash
git clone --branch v$VERSION https://github.com/cryptosharks131/lndg.git && cd lndg
```

* Verify the release

```bash
git verify-commit $(git rev-parse HEAD)
```

**Example** of expected output:

```
gpg: Signature made Mon Dec  9 18:16:45 2024 UTC
gpg:                using RSA key B5690EEEBB952194
gpg: Good signature from "GitHub <noreply@github.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 9684 79A1 AFF9 27E3 7D1A  566B B569 0EEE BB95 2194
```

## Initialization

* Create a Python virtual environment

```bash
python3 -m venv .venv
```
* Install required dependencies

```bash
.venv/bin/python -m pip install -r requirements.txt
```
* Initialize necessary settings for your Django site. A ***FIRST TIME LOGIN PASSWORD*** will be generated - save it somewhere safe.

```bash
.venv/bin/python initialize.py
```

Expected output:

```
Setting up initial user...
Superuser created successfully.
FIRST TIME LOGIN PASSWORD:abc...123
```

## Configure LNDg to use PostgreSQL

### Migrate Django default Superuser and database backend

* As the user `lndg`, enter the LNDg installation folder

```bash
cd ~/lndg
```

* Prepare initialized database for migration

```bash
.venv/bin/python manage.py dumpdata > db.json
```

### Build psycopg2 for PostgreSQL connection

* Upgrade setuptools

```bash
.venv/bin/python -m pip install --upgrade setuptools
```

* Build it

```bash
.venv/bin/python -m pip install psycopg2
```

### Update the LNDg settings

By default, LNDg stores the LN node routing statistics and settings in a SQLite database. We'll update the `DATABASES` section of the initialized `settings.py` file to use our newly created `lndg` database from the [Create PostgreSQL database](#create-postgresql-database-for-lndg) section.

* Change to LNDg configuration folder

```bash
cd ~/lndg/lndg
```

* Edit the `settings.py` file

```bash
nano +87 settings.py --linenumbers
```

* Replace the `DATABASES` section with the following configuration. Save and exit.

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'lndg',
        'USER': 'admin',
        'PASSWORD': 'admin',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

Change back to the LNDg installation folder

```bash
cd ~/lndg
```

### Initialize the PostgreSQL database

```bash
.venv/bin/python manage.py makemigrations && .venv/bin/python manage.py migrate
```

* Load the initial data into the PostgreSQL database *SKIP THIS STEP IF CONNECTING TO EXISTING DATABASE*

```bash
.venv/bin/python manage.py loaddata db.json
```

* Exit to the `admin` user session

```bash
exit
```

## Backend Controller

The LNDg Python script `~/lndg/controller.py` orchestrates the services needed to update the backend database with the most up-to-date information, rebalance any channels based on your LNDg dashboard settings, listen for any failure events in your HTLC stream, and serves the p2p trade secrets.

### Create lndg-controller systemd service

* As user `admin`, create a systemd service file to run the LNDg `controller.py` Python script

```bash
sudo nano /etc/systemd/system/lndg-controller.service
```
* Paste the following configuration. Save and exit.

```
# MiniBolt: systemd unit for LNDg backend controller
# /etc/systemd/system/lndg-controller.service

[Unit]
Description=LNDg backend controller
Requires=lnd.service
After=lnd.service

[Service]
Environment=PYTHONUNBUFFERED=1
ExecStart=/home/lndg/lndg/.venv/bin/python /home/lndg/lndg/controller.py
WorkingDirectory=/home/lndg/lndg

User=lndg
Group=lndg

# Process management
####################
Restart=always
Type=simple
RestartSec=60
TimeoutSec=300
TimeoutStopSec=3

# Hardening Measures
####################
PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true
PrivateDevices=true

[Install]
WantedBy=multi-user.target
```

### Run

* Start the LNDg controller service

```bash
sudo systemctl start lndg-controller
```

* Enable autoboot **(optional)**

```bash
sudo systemctl enable lndg-controller
```

### Validation

* Check the status of the LNDg controller service

```bash
sudo systemctl status lndg-controller
```

## Web Server - uWSGI

[uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/) is an application server that helps deploy web applications, especially those written in Python. It acts as a bridge between web servers (like Nginx) and web application frameworks (like Django).

### Installation

* Change back to the `lndg` user

```bash
sudo su - lndg
```

* Change to the LNDg installation directory

```bash
cd ~/lndg
```

* Install uWSGI within the LNDg Python virtual environment

```bash
.venv/bin/python -m pip install uwsgi
```

### Configuration

* Create the initialization file

```bash
nano lndg.ini
```

* Paste the following configuration. Save and exit

```
# lndg.ini file
[uwsgi]

###########################
# Django-related settings #
###########################
# the base directory (full path)
chdir           = /home/lndg/lndg
# Django's wsgi file
module          = lndg.wsgi
# the virtualenv (full path)
home            = /home/lndg/lndg/.venv
#location of log files
logto           = /var/log/uwsgi/%n.log

############################
# process-related settings #
############################
# master
master          = true
# maximum number of worker processes
processes       = 1
# the socket (use the full path to be safe)
socket          = /home/lndg/lndg/lndg.sock
# ... with appropriate permissions - may be needed
chmod-socket    = 660
# clear environment on exit
vacuum          = true
```

* Create the uwsgi parameter file

```bash
nano uwsgi_params
```

* Paste the following configuration. Save and exit

```
uwsgi_param  QUERY_STRING       $query_string;
uwsgi_param  REQUEST_METHOD     $request_method;
uwsgi_param  CONTENT_TYPE       $content_type;
uwsgi_param  CONTENT_LENGTH     $content_length;

uwsgi_param  REQUEST_URI        "$request_uri";
uwsgi_param  PATH_INFO          "$document_uri";
uwsgi_param  DOCUMENT_ROOT      "$document_root";
uwsgi_param  SERVER_PROTOCOL    "$server_protocol";
uwsgi_param  REQUEST_SCHEME     "$scheme";
uwsgi_param  HTTPS              "$https if_not_empty";

uwsgi_param  REMOTE_ADDR        "$remote_addr";
uwsgi_param  REMOTE_PORT        "$remote_port";
uwsgi_param  SERVER_PORT        "$server_port";
uwsgi_param  SERVER_NAME        "$server_name";
```

* Exit the "lndg" user session

```bash
exit
```

* With user `admin`, create the log and socket files
<!--- need to optimize/describe this workflow for enduser --->

```bash
sudo mkdir /var/log/uwsgi && sudo touch /var/log/uwsgi/lndg.log && sudo chgrp -R www-data /var/log/uwsgi && sudo chmod 660 /var/log/uwsgi/lndg.log
```  

```bash
sudo touch /home/lndg/lndg/lndg.sock && sudo chown lndg:www-data /home/lndg/lndg/lndg.sock && sudo chmod 660 /home/lndg/lndg/lndg.sock
```

### Create uWSGI systemd service

* Create the uWSGI service file

```bash
sudo nano /etc/systemd/system/uwsgi.service
```

* Paste the following configuration. Save and exit

```
# MiniBolt: systemd unit for LNDg uWSGI app
# /etc/systemd/system/uwsgi.service
[Unit]
Description=LNDg uWSGI app
After=lnd.service
Requires=lnd.service

[Service]
ExecStart=/home/lndg/lndg/.venv/bin/uwsgi --ini /home/lndg/lndg/lndg.ini
User=lndg
Group=www-data
Restart=on-failure
# Wait 4 minutes before starting to give LND time to fully start.  Increase if needed.
TimeoutStartSec=240
RestartSec=30
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all

[Install]
WantedBy=multi-user.target
```

* Enable autoboot **(optional)**

```bash
sudo systemctl enable uwsgi
```

## Reverse proxy

In the [security](../index-1/security.md#prepare-nginx-reverse-proxy) section, we set up Nginx as a reverse proxy. Now we can add the LNDg configuration.

* With user `admin`, create the reverse proxy configuration

```bash
sudo nano /etc/nginx/sites-available/lndg-reverse-proxy.conf
```

* Paste the following configuration lines. Save and exit.
```
upstream django {
  server unix:///home/lndg/lndg/lndg.sock; # for a file socket
}

server {
  # the port your site will be served on
  listen 8889 ssl;
  listen [::]:8889 ssl;
  ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
  ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
  ssl_session_timeout 4h;
  ssl_protocols TLSv1.3;
  ssl_prefer_server_ciphers on;
  
  # the domain name it will serve for
  server_name _;
  charset     utf-8;
  
  # max upload size
  client_max_body_size 75M;

  # max wait for django time
  proxy_read_timeout 180;

  # Django media
  location /static {
    alias /home/lndg/lndg/gui/static; # your Django project's static files - amend as required
  }

# Finally, send all non-media requests to the Django server.
  location / {
    uwsgi_pass  django;
    include     /home/lndg/lndg/uwsgi_params; # the uwsgi_params file
  }
}
```
* Create a symlink in the `sites-enabled` directory

```bash
sudo ln -s /etc/nginx/sites-available/lndg-reverse-proxy.conf /etc/nginx/sites-enabled/
```

* Open the nginx configuration file

```bash
sudo nano /etc/nginx/nginx.conf
```

* Add the following configuration lines inside the `http` block, before the closing **`}`**, while taking note of proper indents. Save and exit.

```
  # settings used for LNDg Django site
  include /etc/nginx/mime.types;
  default_type application/octet-stream;
```

* Test the nginx configuration

```bash
sudo nginx -t
```

Expected output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

* Reload NGINX configuration to apply changes

```bash
sudo systemctl reload nginx
```
## Run the web app

* Start the uWSGI service

```bash
sudo systemctl start uwsgi
```

* Check the status of the uWSGI service

```bash
sudo systemctl status uwsgi
```

Expected output:

```
> Starting LNDg uWSGI app...
> [uWSGI] getting INI configuration from /home/lndg/lndg/lndg.ini
> Started LNDg uWSGI app.

```

### Validation

* To check the uwsgi log (Ctrl+c to exit the log)

```bash
sudo journalctl -fu uwsgi
```

## First time login

You can now access LNDg from within your local network by browsing to https://minibolt.local:8889 (or your equivalent IP address).

* The default login is `lndg-admin` and the first time password was generated by the `initialize.py` script in the [initialization](lndg.md#initialization) step. If you forgot or didn't write it down, you can find the password at `/home/lndg/lndg/data/password.txt` (you should delete this file for security reasons).
* *You can change the initial password after logging into in the GUI.*

* To delete the first time password file

```bash
sudo rm /home/lndg/lndg/data/password.txt
```

### Changing the initial password

After you log in, you can change by password by navigating to https://minibolt.local:8889/lndg-admin/ (or your equivalent IP address) and clicking on `CHANGE PASSWORD` in the top right.

## Dashboard privacy configuration

LNDg offers the possibility to create links to blockchain and lightning explorers on node aliases and transaction IDs. By default, LNDg uses public website *mempool.space* for both explorers.

### Blockchain explorer

To preserve privacy it is better that you use your own self-hosted blockchain explorer (e.g. the BTC RPC Explorer).

* Open your LNDg website at https://minibolt.local:8889 (replace minibolt.local with your node's IP address if necessary)
* Click on the `Advanced Settings` link
* Scroll down to the `Update Local Settings` section
* Find the `NET URL` option and paste the following value if you use [BTC RPC Explorer](../../bitcoin/btcrpcexplorer.md):

```
https://minibolt.local:4000
``` 
<!--->
### Lightning explorer

Although as of this writing there is not yet a self-hosted, private, lightning explorer, the Mempool lightning explorer offers a better lightning explorer than 1ML and is probably less likely to log your IP address.

* Find the `Graph URL` option
  * if you don't want to leak your IP address, delete the content of the box and leave it empty
  * if you want to use Mempool, enter: `https://mempool.space/lightning`. As an additional privacy step, you may consider running through a VPN.
--->

## Extras (optional)

### Remote access over Tor 

* With user `admin`, edit the `torrc` file

```bash
sudo nano +63 /etc/tor/torrc --linenumbers
```

* Add the following three lines in the “location-hidden services” section, below `## This section is just for location-hidden services ##`. Save and exit

```
# Hidden service LNDg
HiddenServiceDir /var/lib/tor/hidden_service_lndg/
HiddenServiceVersion 3
HiddenServicePoWDefensesEnabled 1
HiddenServicePort 443 127.0.0.1:8889
```

* Reload Tor to apply changes

```bash
sudo systemctl reload tor
```

* Get your Onion address

```bash
sudo cat /var/lib/tor/hidden_service_lndg/hostname
```

Expected output:

```
abcdefg..............xyz.onion
```

* With the [Tor browser](https://www.torproject.org), you can access this onion address from any device

## Upgrade

* Change to the `lndg` user

```bash
sudo su - lndg
```

* Go to the LNDg installation directory

```bash
cd ~/lndg
```

* Set the environment variable to the new version

```bash
VERSION=1.9.1
```

* Pull the changes from GitHub

```bash
git pull v$VERSION 
```

* Migrate any database changes

```bash
.venv/bin/python manage.py migrate
```

* Restart the uWSGI service

```bash
sudo systemctl restart uwsgi
``` 

## Uninstall

* Stop the `uwsgi` systemd service

```bash
sudo systemctl stop uwsgi.service
```

* Disable the `uwsgi` systemd service

```bash
sudo systemctl disable uwsgi.service
  ```

* Delete the `uwsgi` systemd service file

```bash
sudo rm /etc/systemd/system/uwsgi.service
```

* Delete the uwsgi log file

```bash
sudo rm /var/log/uwsgi/lndg.log
```

* Stop the `lndg-controller` systemd service

```bash
sudo systemctl stop lndg-controller.service
```

* Disable the `lndg-controller` systemd service

```bash
sudo systemctl disable lndg-controller.service
```

* Delete the `lndg-controller` systemd service file

```bash
sudo rm /etc/systemd/system/lndg-controller.service
```

### Delete the user & group

* Delete the `lndg` user. Do not worry about the `userdel: mempool mail spool (/var/mail/lndg) not found`

```bash
sudo userdel -rf lndg
```

### Uninstall Tor hidden service

* Comment or remove the LNDg hidden service lines from the `torrc` file. Save and exit

```bash
sudo nano +63 /etc/tor/torrc --linenumbers
```

```
# Hidden service LNDg
#HiddenServiceDir /var/lib/tor/hidden_service_lndg/
#HiddenServiceVersion 3
#HiddenServicePoWDefensesEnabled 1
#HiddenServicePort 443
```

* Reload Tor to apply changes

```bash
sudo systemctl reload tor
```

### Uninstall reverse proxy and firewall configurations

* Ensure you are logged in as the `admin` user, delete the LNDg reverse proxy configuration file

```bash
sudo rm /etc/nginx/sites-available/lndg-reverse-proxy.conf
```

* Delete the symlink in the `sites-enabled` directory

```bash
sudo rm /etc/nginx/sites-enabled/lndg-reverse-proxy.conf
```

* Delete or comment out the associated HTTP server block lines from the `nginx.conf` file (unless used for another service)

```bash
sudo nano /etc/nginx/nginx.conf
```

* Remove or comment the following lines. Save and exit.

```
  # settings used for LNDg Django site
  include /etc/nginx/mime.types;
  default_type application/octet-stream;
```

* Test the nginx configuration

```bash
sudo nginx -t
```

Expected output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

* Reload NGINX configuration to apply changes

```bash
sudo systemctl reload nginx
```

* Display the UFW firewall rules and note the number(s) of the rule(s) for LNDg (e.g. X below)
  
```bash
sudo ufw status numbered
```

Expected output:

```
[X] 8889/tcp                   ALLOW IN    Anywhere                   # allow LNDg SSL from anywhere
```

* Delete the LNDg rule(s) (check that the rule to be deleted is the correct one and type “y” and “Enter” when prompted)

```bash
sudo ufw delete X
```

### Delete PostgreSQL database

* With user `admin`, delete the PostgreSQL database

```bash
sudo -u postgres dropdb lndg
```

<!---
<< Back: [+ Lightning](index.md)
--->
