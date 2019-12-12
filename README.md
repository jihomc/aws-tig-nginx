# aws-tig-nginx 





How to set up an Amazon aws ec2 instance with the Telegraf, InfluxDB, and Grafana (TIG) stack for system monitoring dashboards, with an Nginx reverse proxy in front to enable HTTPS access via the web from my personal domain.




## Getting Started



### Prerequisites



The following dependencies are required:

* Amazon aws ec2 running Ubuntu 18.04 LTS (Bionic)
* A registered domain with DNS management - DigitalOcean 
* InfluxDB
* Telegraf
* Grafana
* Nginx
* Openssl

This setup uses DigitalOcean DNS to direct requests to www.mydomain.com to the aws ec2 server.   
add aws security groups



## Installations




### InfluxDB



#### Installation

Install packages and start InfluxDB service

https://portal.influxdata.com/downloads/

```shell
wget https://dl.influxdata.com/influxdb/releases/influxdb_1.7.9_amd64.deb
sudo dpkg -i influxdb_1.7.9_amd64.deb

sudo systemctl unmask influxdb.service
sudo systemctl start influxdb
```


#### Configuration


Command to view the default configuration

```shell
influxdb config
```

InfluxDB local configuration file

```shell
/etc/influxdb/influxdb.conf
```

Point the process to the configuration file

```shell
influxd -config /etc/influxdb/influxdb.conf
```


#### Configuration 1 - HTTP authentication


InfluxDB - How to configure accounts and enable HTTP authentication

https://docs.influxdata.com/influxdb/v1.7/administration/authentication_and_authorization/#authorization

1. Create admin account and telegraf user account in influxDB

```shell
influx
CREATE USER username WITH PASSWORD 'password' WITH ALL PRIVILEGES
SHOW USERS
SHOW GRANTS for user_name
```

2. Enable HTTP authentication in the configuration file

/etc/influxdb/influxdb.conf

```shell
[http]
        enabled = true
        auth-enabled = true
        bind-address = ":8086"
```
 

#### Configuration 2 - HTTPS w/ self-signed certificates

InfluxDB - How to enable HTTPS and connect Telegraf to InfluxDB

1. Create self-signed certificates for InfluxDB

```shell
sudo openssl req -x509 -nodes -newkey rsa:2048 -keyout /etc/ssl/influxdb-selfsigned.key -out /etc/ssl/influxdb-selfsigned.crt -days 365
```

2. Grant InfluxDB permissions on the certificate

```shell
sudo chown influxdb:influxdb /etc/ssl/influxdb-selfsigned.*
```

3. Enable HTTPS in the configuration file

/etc/influxdb/influxdb.conf

```shell
https-enabled = true
https-certificate = "/etc/ssl/influxdb-selfsigned.crt"
https-private-key = "/etc/ssl/influxdb-selfsigned.key"
```

4. Restart the InfluxDB service

```shell
systemctl restart influxdb
```

5. Accessing the InfluxDB CLI - HTTPS Authentication - (localhost)

```shell
influx -ssl -unsafeSsl -host 127.0.0.1
> auth
> username
> password
```



### Telegraf



#### Installation


Install packages and enable Telegraf service

```shell
wget https://dl.influxdata.com/telegraf/releases/telegraf_1.10.3-1_amd64.deb
sudo dpkg -i telegraf_1.10.3-1_amd64.deb

sudo systemctl start telegraf
sudo systemctl enable telegraf
```


#### Configuration 1 - HTTP


Edit Telegraf config to authenticate requests to InfluxDB

vim /etc/telegraf/telegraf.conf

```shell
# Configuration for sending metrics to InfluxDB
[[outputs.influxdb]]

## The full HTTP or UDP URL for your InfluxDB instance.
urls = ["http://127.0.0.1:8086"]

## HTTP Basic Auth
username = "username"
password = "password"

```

Check for errors when restarting telegraf service

```shell
$ sudo journalctl -f -u telegraf.service
```


#### Configuration 2 - HTTPS


Connect Telegraf to InfluxDB via HTTPS w/ self-signed certificates

vim /etc/telegraf/telegraf.conf

```shell
# Configuration for sending metrics to InfluxDB
[[outputs.influxdb]]

## The full HTTP or UDP URL for your InfluxDB instance.
urls = ["https://127.0.0.1:8086"]

## Optional TLS Config for use on HTTP connections.
## Use TLS but skip chain & host verification
insecure_skip_verify = true

```

Restart Telegraf

```shell
systemctl restart telegraf
```



### Grafana-Server



#### Installation


1. Update and install repository (for stable releases)

```shell
sudo apt-get install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
```

2. Add the gpg-key

```shell
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

3. Update Apt repositories and install Grafana

```shell
sudo apt-get update
sudo apt-get install grafana
```

4. Start the server

```shell
systemctl daemon-reload
systemctl start grafana-server
systemctl status grafana-server
```

5. Enable the systemd service so Grafana starts at boot

```shell
sudo systemctl enable grafana-server.service
```


#### Optional Configuration - HTTPS

How to enable HTTPS for Grafana. Not necessary because we will use Nginx with HTTPS to access the Grafana server.

Create self-signed certificates for Grafana

```shell
sudo openssl req -x509 -nodes -newkey rsa:2048 -keyout /etc/ssl/grafana-selfsigned.key -out /etc/ssl/grafana-selfsigned.crt -days 365
```

Grant Grafana permissions on the certificate

```shell
sudo chown grafana:grafana /etc/ssl/grafana-self-signed.*
```

Edit [server] protocol, certs & key file in Grafana config

vim /etc/grafana/grafana-server.ini

```shell
[server]

# Protocol (http, https, h2, socket)
protocol = https

# https certs & key file
cert_file = /etc/ssl/grafana-selfsigned.crt
cert_key = /etc/ssl/grafana-selfsigned.key
```

Grafana logs

```shell
vim /var/log/grafana/grafana.log
cat /var/log/grafana/grafana.log | grep eror | less
```


#### Usage


Accessing the Grafana dashboard

http://127.0.0.1:3000



### Nginx 



#### Installation - Nginx


How to set up an Nginx reverse proxy to Grafana server with HTTPS enabled

With Nginx, there is no need to enable HTTPS directly in Grafana

Install Nginx

```shell
sudo apt install curl gnupg2 ca-certificates lsb-release
echo "deb http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo apt-key add -
sudo apt-key fingerprint ABF5BD827BD9BF62
sudo apt update
sudo apt install nginx
sudo systemctl start nginx
```

Nginx configuration files

vim /etc/nginx/nginx.conf
vim /etc/nginx/sites-available/default
vim /etc/nginx/site-enabled/default


#### Installation - Certbot


Use Certbot to install Lets Encrypt certificate to enable HTTPS

Certbot installation -  Nginx Ubuntu 18.04 LTS (bionic)

```shell
https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx
sudo certbot --nginx
```


#### Configuration - HTTPS


Modify Nginx configuration file to meet the recommended Mozilla ssl-config. Refer to nginx/nginx.conf file for all configuration details.


##### Mozilla ssl-config


https://ssl-config.mozilla.org/


##### Certbot docs - where are the certificates?

https://certbot.eff.org/docs/using.html#where-are-my-certificates

/etc/letsencrypt/live/$domain/certificate.pem


##### Add certificates to Nginx configuration


```shell
vim /etc/nginx/nginx.conf
```

```shell
# Nginx ssl_certificate_key
ssl_certificate /etc/letsencrypt/live/$domain/fullchain.pem;

# Nginx ssl_certificate
ssl_certificate_key /etc/letsencrypt/live/$domain/privkey.pem;

# Nginx ssl_trusted_certificate (if using OCSP Stapling)
ssl_trusted_certificate /etc/letsencrypt/live/$domain/chain.pem;
```
Look up the IP address of your resolver

```shell
sudo vim /etc/resolv.conf
```

nginx.conf

```shell
# replace with the IP address of your resolver
resolver 127.0.0.53 [::1]:5353 valid=30s;
```


#### Configuration - Nginx reverse proxy


How to set up Nginx to reverse proxy the Grafana server with HTTPS


##### View nginx/nginx.conf to see complete file 


Nginx server configuraton


##### Listen on port 80 for requests to my domain and redirect to HTTPS


```shell
server {

                listen 80;
                listen [::]:80;

                server_name mydomain.com www.mydomain.com;
                return 301 https://$server_name$request_uri;
        }
```


##### Listen on port 443 for requests

SSL configuration will be applied and HTTPS will be enabled before passing the request to the proxied Grafana server. As a result, HTTPS does not need to be configured for Grafana directly.


```shell
server {

        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name mydomain.com www.mydomain.com;
	
 	# Enter SSL configuration here

 	access_log /var/log/nginx/grafana-access.log;
        error_log /var/log/nginx/grafana-error.log;

        location / {
                 proxy_set_header Host $http_host;
                 proxy_set_header X-Real-IP $remote_addr;
                 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                 proxy_set_header X-Forwarded-Proto $scheme;
                 proxy_pass http://127.0.0.1:3000;
        }
}
```


#### Testing - Qualys SSL Server Test


https://www.ssllabs.com/ssltest/

