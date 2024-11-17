---
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
---

# LNDg

[LNDg](https://github.com/cryptosharks131/lndg){:target="_blank"} is a lite GUI web interface to help you manually manage your node and automate operations such as rebalancing, fee adjustments and channel node opening.

{% hint style="danger" %}
Difficultry: Hard
{% endhint %}

<figure><img src="../../.gitbook/assets/lndg.png" alt=""><figcaption></figcaption></figure>

## Requirements

* [Bitcoin Core](../../bitcoin/bitcoin/bitcoin-client.md)
* [LND](../../lightning/lightning-client.md) 
* Others
  * [PostgreSQL](../system/postgresql.md)
  * [virtualenv](https://virtualenv.pypa.io/en/latest/)
  * [uWSGI](https://uwsgi-docs.readthedocs.io/)

## Preparations

To run LNDg you will need to install `PostgreSQL`, `virtualenv`, and `uWSGI` (within the Python virtual environment created with virtualenv).

### Firewall

* Configure firewall to allow incoming HTTP requests:

```bash
sudo ufw allow 8889/tcp comment 'allow LNDg ssl from anywhere'
```

### PostgreSQL Database for LNDg

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

#### Create PostgreSQL database

* With user `admin`, create a new database for LNDg as the `postgres` user and assign the user `admin` as the owner

```bash
sudo -u postgres createdb -O admin lndg
```
* Get required packages to connect LNDg with PostgreSQL

```bash
sudo apt install gcc python3-dev libpq-dev
```

### Python virtual environment

[Virtualenv](https://virtualenv.pypa.io/en/latest/) is a tool to create isolated Python environments. 

* With user `admin`, check if you already have `virtualenv` installed

```bash
virtualenv --version
```

**Example** of expected output:

```bash
virtualenv 20.13.0+ds from /usr/lib/python3/dist-packages/virtualenv/__init__.py
```

{% hint style="info" %}
If you obtain "**command not found**" outputs, install `virtualenv` with apt
{% endhint %}

```bash
sudo apt install virtualenv
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
* Add 'www-data' user to the lndg group

```bash
sudo adduser www-data lndg
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

<pre><code>lrwxrwxrwx 1 lndg lndg 9 Nov 10 01:03 <a data-footnote-ref href="#user-content-fn-10">.lnd -> /data/lnd</a></code></pre>

<!--- OLD STYLE

* Check the version number of the latest LNDg release (you can also confirm with the [release page](https://github.com/cryptosharks131/lndg/releases))

```bash
LATEST_RELEASE=$(wget -qO- https://api.github.com/repos/cryptosharks131/lndg/releases/latest | grep -oP '"tag_name":\s*"\K([^"]+)') && echo $LATEST_RELEASE
```  

**Example** of expected output:

```
v1.9.0
```
--->

* Set a temporary version environment variable to the installation

```bash
VERSION=1.9.0
```

* Immport the GPG key of the developer

```bash
curl https://github.com/cryptosharks131.gpg | gpg --import
```

* Download the source code directly from GitHub, select the latest release branch associated, and go to the `lndg` folder

```bash
git clone --branch v$VERSION https://github.com/cryptosharks131/lndg.git && cd lndg
```
<!---

## will include in when developer renews key ##

* Verify the release

```bash
git verify-commit v$VERSION
```

**Example** of expected output:

```
gpg: Signature made Tue Sep 24 15:58:10 2024 UTC
gpg:                using RSA key 1957FD54782C190096F4166F0A50748567ADEB28
gpg: Good signature from "cryptosharks131 <cryptosharks131@gmail.com>" [expired]
gpg: Note: This key has expired!
Primary key fingerprint: 1957 FD54 782C 1900 96F4  166F 0A50 7485 67AD EB28
```
--->

* Create a Python virtual environment

```bash
virtualenv -p python3 .venv
```

* Install required dependencies

```bash
.venv/bin/python -m pip install -r requirements.txt
```
* Initialize necessary settings for your Django site. A first time password will be generated - save it somewhere safe.

```bash
.venv/bin/python initialize.py
```

Expected output:

```
Setting up initial user...
Superuser created successfully.
FIRST TIME LOGIN PASSWORD:abc...123
```

<!--- Original test for debug server - thinking this step is unneccessary

### First start

* Still with the "lndg" user, start the server

```bash
.venv/bin/python manage.py runserver 0.0.0.0:8889
```
 
Expected output

```
> [...]
> Starting development server at http://0.0.0.0:8889/
> Quit the server with CONTROL-C.
```

* Now point your browser to the LNDg Python server, for example http://minibolt.local:8889 
(or your node's IP address, e.g. http://192.168.0.20:8889). 

* The initial login user is "lndg-admin" and the password is the one generated just above. 
If you didn't save the password, you can get it again with: `nano /home/lndg/lndg/data/lndg-admin.txt`

* Shut down the server with `Ctrl+c`

* For extra security, delete the text file that contains the password

```bash
rm /home/lndg/lndg/data/lndg-admin.txt
```
 
* Exit `lndg` user session and return to `admin` for the next steps.

```bash
exit
```

--->
### Configure LNDg to use PostgreSQL

* As the user `lndg`, enter the LNDg installation folder

```bash
cd ~/lndg
```

* Upgrade setuptools

```bash
.venv/bin/python -m pip install --upgrade setuptools
```

* Build psycopg2 for PostgreSQL connection

```bash
.venv/bin/python -m pip install psycopg2
```

By default, LNDg stores the LN node routing statistics and settings in a SQLite database. We'll update the `DATABASES` section of the initialized `settings.py` file to use our newly created `lndg` database.

* Change to LNDg configuration folder

```bash
cd ~/lndg/lndg
```

* Edit the `settings.py` file

```bash
nano +87 settings.py --linenumbers
```

* Replace the `DATABASES` section with the following:

<pre></code>
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
</code></pre>

Change back to the LNDg installation folder

```bash
cd ~/lndg
```

* Initialize the PostgreSQL database

```bash
.venv/bin/python manage.py migrate
```

* Exit to the `admin` user session

```bash
exit
```

### Backend Controller

The LNDg Python script `~/lndg/controller.py` orchestrates the services needed to update the backend database with the most up-to-date information, rebalance any channels based on your LNDg dashboard settings, listen for any failure events in your HTLC stream, and serves the p2p trade secrets.

* As user `admin`, create a systemd service file to run the LNDg `controller.py` Python script

```bash
sudo nano /etc/systemd/system/lndg-controller.service
```
* Paste the following configuration. Save and exit.

<pre><code>
# MiniBolt: systemd unit for LNDg backend controller
# /etc/systemd/system/lndg-controller.service
  
[Unit]
Description=LNDg backend controller
<strong>Requires=lnd.service
</strong>After=lnd.service

[Service]
WorkingDirectory=/home/lndg/lndg
Environment=PYTHONBUFFERED=1
ExecStart=/home/lndg/lndg/.venv/bin/python /home/lndg/lndg/controller.py

User=lndg
Group=lndg

# Process management
####################
Restart=always
Type=notify
RestartSec=60
TimoutSec=300

# Hardening Measures
####################
PrivateTmp=true
ProtectSystem=full
NoNewPriviledges=true
PrivateDevices=true

[Install]
WantedBy=multi-user.target
</code></pre>

* Enable autoboot **(optional)**

```bash
sudo systemctl enable lndg-controller
```
### Web server configuration

uWSGI is an application server that helps deploy web applications, especially those written in Python. It acts as a bridge between web servers (like Nginx) and web application frameworks (like Django).

* Change to the `lndg` user

```bash
sudo su - lndg
```

* Install uWSGI within the LNDg Python virtual environment

```bash
.venv/bin/python -m pip install uwsgi
```

* Create the initialization file

```bash
nano lndg.ini
```

* Paste the following configuration. Save and exit

<pre><code>
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
</code></pre>

* Create the uwsgi parameter file

```bash
nano uwsgi_params
```

* Paste the following configuration. Save and exit

<pre><code>
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
</code></pre>

* Exit the "lndg" user session

```bash
exit
```

* Create the log and socket files
<!--- not sure how to optimize/describe this workflow for enduser --->

```bash
sudo mkdir /var/log/uwsgi && sudo touch /var/log/uwsgi/lndg.log && sudo chgrp -R www-data /var/log/uwsgi && sudo chmod 660 /var/log/uwsgi/lndg.log
```  

```bash
sudo touch /home/lndg/lndg/lndg.sock && sudo chown lndg:www-data /home/lndg/lndg/lndg.sock && sudo chmod 660 /home/lndg/lndg/lndg.sock
```

* Create the uWSGI service file

```bash
sudo nano /etc/systemd/system/uwsgi.service
```

* Paste the following configuration. Save and exit

<pre><code>
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
</code></pre>

* Enable autoboot **(optional)**

```bash
sudo systemctl enable uwsgi
```

### Reverse proxy

In the security [section](../index-1/security.md#prepare-nginx-reverse-proxy), we set up Nginx as a reverse proxy. Now we can add the LNDg configuration.

* With user `admin`, create the reverse proxy configuration

```bash
sudo nano /etc/nginx/sites-available/lndg-reverse-proxy.conf
```

* Paste the following configuration lines. Save and exit.

<pre><code>
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
</pre></code>

* Create a symlink in the `sites-enabled` directory

```bash
sudo ln -s /etc/nginx/sites-available/lndg-reverse-proxy.conf /etc/nginx/sites-enabled/
```

* Open the nginx configuration file

```bash
sudo nano /etc/nginx/nginx.conf
```

* Add the following configuration lines inside the `http` block. Save and exit.

<code></pre>
  # settings used for LNDg Django site
  include /etc/nginx/mime.types;
  default_type application/octet-stream;
</pre></code>

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
### Run the web app

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

* To check the uwsgi log (Ctrl+c to exit the log)

```bash
sudo journalctl -fu uwsgi
```

You can now access LNDg from within your local network by browsing to https://minibolt.local:8889 (or your equivalent IP address).

The default login is `lndg-admin` and the first time password was generated by the `initialize.py` script. You can find the password at `/home/lndg/lndg/data/password.txt'(you should delete this file for security reasons). You can change the password in the GUI.

* To delete the first time password file

```bash
sudo rm /home/lndg/lndg/data/password.txt
```

## Dashboard privacy configuration

LNDg offers the possibility to create links to blockchain and lightning explorers on node aliases and transaction IDs. By default, LNDg uses public websites 1ml.com for its lightning explorer and mempool.space for its blockchain explorer.

### Blockchain explorer

To preserve privacy it is better that you use your own self-hosted blockchain explorer (e.g. the BTC RPC Explorer).

* Open your LNDg website at https://minibolt.local:8889 (replace minibolt.local with your node's IP address if necessary)
* Click on the "Advanced settings" link
* Scroll down to the "Update Local Settings" section
* Find the "NET URL" option and paste the following value if you use [BTC RPC Explorer](../../bitcoin/btcrpcexplorer.md):

```
https://minibolt.local:4000
``` 

### Lightning explorer

Although there is not yet a self-hosted, private, lightning explorer, the Mempool lightning explorer offers a better lightning explorer than 1ML and is probably less likely to log your IP address.

* Find the "Graph URL" option and paste the following value:
  * if you don't want to leak your IP address, delete the content of the box and leave it empty
  * if you want to use Mempool, enter: `https://mempool.space/lightning`. As an additional privacy step, you may consider running through a vpn.




<!--- will finish this section later

## Remote access over Tor (optional) 

Do you want to access LNDg remotely? You can easily do so by adding a Tor hidden service on the RaspiBolt and accessing LNDg with the Tor browser from any device.

* Add the following three lines in the “location-hidden services” section in the `torrc file`. Save and exit.

  ```sh
  $ sudo nano /etc/tor/torrc
  ```

  ```ini
  ############### This section is just for location-hidden services ###
  # Hidden service LNDg
  HiddenServiceDir /var/lib/tor/hidden_service_lndg/
  HiddenServiceVersion 3
  HiddenServicePort 443 127.0.0.1:8889
  ```

* Reload Tor configuration and get your connection address.

  ```sh
  $ sudo systemctl reload tor
  $ sudo cat /var/lib/tor/hidden_service_lndg/hostname
  > abcdefg..............xyz.onion
  ```

With the Tor browser, you can access this onion address from any device.

## For the future: LNDg update

* With user "admin", stop the `uwsgi` systemd service. The other LNDg systemd timers and services will stop automatically.

  ```sh
  $ sudo systemctl stop uwsgi.service
  $ sudo su - lndg
  ```

* Fetch the latest GitHub repository information, display the release tags (use the latest 1.7.1 in this example), and update:

  ```sh
  $ cd /home/lndg/lndg
  $ git fetch
  $ git reset --hard HEAD
  $ git tag
  $ git checkout v1.7.1
  $ .venv/bin/pip install -r requirements.txt
  $ .venv/bin/pip install --upgrade protobuf
  $ rm lndg/settings.py
  $ .venv/bin/python initialize.py -wn
  $ .venv/bin/python manage.py migrate
  $ exit
  ```
 
  
* Start the `uwsgi` systemd service again. The other LNDg timers and services will start automatically.

  ```sh
  $ sudo systemctl start uwsgi.service
  ```

## Uninstall

* Stop and disable the `uwsgi` systemd service

  ```sh
  $ sudo systemctl stop uwsgi.service
  $ sudo systemctl disable uwsgi.service
  ```

* Delete all the LNDg systemd services and timers
 
  ```sh
  $ cd /etc/systemd/system/
  $ sudo rm uwsgi.service jobs-lndg.service rebalancer-lndg.service htlc-stream-lndg.service  
  $ sudo rm jobs-lndg.timer rebalancer-lndg.timer
  $ cd
  ```

* Display the UFW firewall rules and notes the numbers of the rules for Mempool (e.g., X and Y below)
  
  ```sh
  $ sudo ufw status numbered
  > [...]
  > [X] 8889/tcp                   ALLOW IN    Anywhere                   # allow LNDg SSL
  > [...]
  > [Y] 8889/tcp (v6)              ALLOW IN    Anywhere (v6)              # allow LNDg SSL
  ```

* Delete the two LNDg rules (check that the rule to be deleted is the correct one and type “y” and “Enter” when prompted)
  
  ```sh
  $ sudo ufw delete Y
  $ sudo ufw delete X
  ```

* Delete the LNDg nginx configuration file and symlink

  ```sh
  $ sudo rm /etc/nginx/sites-available/lndg-ssl.conf
  $ sudo rm /etc/nginx/sites-enabled/lndg-ssl.conf
  ```

* Delete or comment out the HTTP server block from the `nginx.conf` file (unless you use it for another service, _e.g._ [Mempool](../bitcoin/mempool.md), [Homer](../raspberry-pi/homer.md) etc)

  ```sh
  $ sudo nano /etc/nginx/nginx.conf
  ```

  ```ini
  #http {
  # [...]
  #}
  ```

* Test the nginx configuration & restart nginx

  ```sh
  $ sudo nginx -t
  > nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
  > nginx: configuration file /etc/nginx/nginx.conf test is successful
  $ sudo systemctl reload nginx
  ```

* Delete the uwsgi log file

  ```sh
  $ sudo rm /var/log/uwsgi/lndg.log
  ```

* Delete the “lndg” user. Do not worry about the `userdel: mempool mail spool (/var/mail/lndg) not found`.
 
  ```sh
  $ sudo su -
  $ userdel -r lndg
  > userdel: lndg mail spool (/var/mail/lndg) not found]
  $ exit
  ```

<br /><br />

--->

<< Back: [+ Lightning](index.md)
