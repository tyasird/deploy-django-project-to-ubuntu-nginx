# Deploy Django Project to Ubuntu-Nginx Server

## Installation


#### install python venv and nginx 
```sh
sudo apt update && sudo apt upgrade -y
sudo apt -y install build-essential python3-venv python3-dev libpq-dev nginx
```



#### allow nginx, create project folder and conf file. 
```sh
sudo ufw allow 'Nginx Full'
mkdir -p /var/www/dcna
sudo nano /etc/nginx/sites-available/dcna.conf
sudo ln -s /etc/nginx/sites-available/dcna.conf /etc/nginx/sites-enabled/dcna.conf
sudo adduser django
sudo usermod -aG django www-data
```

#### create user, venv, django and packages
```sh
cd /var/www/dcna
chown -R django:www-data .
sudo su django
python3 -m venv venv
. venv/bin/activate
pip install --upgrade pip
pip install django gunicorn scipy concurrent_log_handler py4cytoscape pandas numpy openpyxl xlsxwriter gevent
zip -r backup.zip .
```

#### create gunicorn service
```sh
sudo nano /etc/systemd/system/gunicorn.service
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn.service
```

#### install SSL
```sh
sudo snap install core; sudo snap refresh core
sudo apt-get remove certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
certbot --nginx --redirect -d dcna.computationalbiology.org -m mail@hotmail.com --agree-tos
#sudo certbot delete --cert-name example.com
```

#### install remote desktop
```sh
sudo apt -y install xrdp
sudo adduser xrdp ssl-cert
sudo ufw allow 3389
sudo apt install xfce4
```

#### install xfce desktop
```sh
change desktop to xfce4
sudo update-alternatives --config x-session-manager
```



# Conf Files


#### nginx conf with gunicorn
```
server {

server_name dcna.computationalbiology.org;

# Process static file requests
location /static/ {
    root /var/www/dcna;

    # Set expiration of assets to MAX for caching
    expires max;
}

location  /uploads/ {
	root /var/www/dcna;
}


# Deny accesses to the virtual environment directory
location /venv {
    return 444;
}

# Pass regular requests to Gunicorn
location / {
    # set the correct HTTP headers for Gunicorn
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;

    # we don't want nginx trying to do something clever with
    # redirects, we set the Host: header above already.
    proxy_redirect off;

    # turn off the proxy buffering to handle streaming request/responses
    # or other fancy features like Comet, Long polling, or Web sockets.
    proxy_buffering off;

    proxy_pass http://127.0.0.1:8000;
}

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/dcna.computationalbiology.org/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/dcna.computationalbiology.org/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = dcna.computationalbiology.org) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


listen 80;
listen [::]:80;

server_name dcna.computationalbiology.org;
    return 404; # managed by Certbot
}
```


#### django settings.py for static files
```
ALLOWED_HOSTS = ['127.0.0.1','IP','DOMAIN','localhost']

STATIC_DIR = os.path.join(BASE_DIR, 'static')
STATIC_URL = '/static/'
STATIC_ROOT = '/var/www/dcna/static/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'uploads').replace('\\', '/')
MEDIA_URL = '/uploads/'

>django urls.py for static files
urlpatterns = [
    ...
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT) + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```



#### domain.conf
```
server {
	listen 80;
	server_name computationalbiology.org www.computationalbiology.org;
	root /var/www/cbl;

	location / {
	   try_files $uri $uri/ =404;
	}
}
````

#### gunicorn.py (config file for gunicorn)
```
command = '/var/www/dcna/venv/bin/gunicorn'
pythonpath = '/var/www/dcna/venv/bin'
bind = '127.0.0.1:8000'
workers = 3
accesslog ='/var/www/dcna/logs/access.log'
errorlog='/var/www/dcna/logs/error.log'
timeout= 1200
worker_class = 'gevent' 
log_level = 'info'
```

#### uWsgi

```sh
pip3 install uwsgi
nano conf/uwsgi_params
uwsgi  --uid www-data --gid www-data --ini conf/uwsgi.ini
(sudo pkill -f uwsgi -9)

sudo mkdir -p env/vassals
sudo ln -s conf/uwsgi.ini env/vassals/
uwsgi --emperor /var/www/dcna/env/vassals 
```

#### uwsgi.ini

```
[uwsgi]
# full path to Django project's root directory
chdir            = /var/www/dcna/
# Django's wsgi file
module           = config.wsgi
# full path to python virtual env
home             = /var/www/dcna/env
# enable uwsgi master process
master          = true
# maximum number of worker processes
processes       = 10
# the socket (use the full path to be safe
socket          = /var/www/dcna/dcna.sock

# socket permissions
chmod-socket    = 666
# clear environment on exit
vacuum          = true
# daemonize uwsgi and write messages into given log
daemonize       = /var/www/dcna/logs/uwsgi.log

harakiri = 1200
harakiri-verbose = true
post-buffering = 1
socket-timeout = 600
#max-worker-lifetime = 35
```


#### nginx conf for uWsgi & django
```
upstream django {
    server unix:///var/www/dcna/dcna.sock fail_timeout=600s; 
}

server {
    listen 80;
	client_max_body_size 100M;
    server_name dcna.computationalbiology.org;
	
	client_body_timeout   600;
	client_header_timeout 600;

    location /static {
        alias   /var/www/dcna/static;
    }
	
	location  /uploads {
        alias /var/www/dcna/uploads;
    }

    location / {
		proxy_connect_timeout   600;
		proxy_send_timeout      600;
		proxy_read_timeout      600;
		keepalive_timeout     600;
		uwsgi_read_timeout 18000;
		uwsgi_pass  django;
        include     /etc/nginx/uwsgi_params;
   }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/computationalbiology.org/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/computationalbiology.org/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
```

#### nginx conf for uWsgi & Flask
```
server {
	listen 80;
	server_name 159.65.204.133;
	client_max_body_size 100M;


	location /flask {
		alias /var/www/flask;
		proxy_pass http://159.65.204.133:5000;
	}

	location /ml_train {
		alias /var/www/ml_train;
		proxy_pass http://159.65.204.133:4000;
	}
	
}
```
