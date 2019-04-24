# How to set up HAProxy with autorenewing Let's Encrypt certs

These instructions assume you are running on Ubuntu 18.04.1 LTS


## Step 1 - Install Let's Encrypt `certbot`

```
add-apt-repository ppa:certbot/certbot
apt-get update
apt-get install certbot
```


## Step 2 - Obtain a Let's Encrypt cert

We will use `certbot` to get a cert. You will need to run this step before running `haproxy` (or you will need to temporarily stop `haproxy`), since `certbot` will bind to port 80 in this step.

These instructions show you how to obtain a certificate for two domains: `example.com` and `www.example.com`:

```
sudo certbot certonly --standalone -d example.com,www.example.com
```

Now, you should be able to run the command `certbot certificates` and see the cert you just obtained:

```
# certbot certificates
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Found the following certs:
  Certificate Name: example.com
    Domains: example.com, www.example.com
    Expiry Date: 2019-01-01 00:00:00+00:00 (VALID: 30 days)
    Certificate Path: /etc/letsencrypt/live/example.com/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/example.com/privkey.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```



## Step 3 - Install HAProxy

```
apt-get install haproxy
```


## Step 4 - Combine certs to create a combined .pem file

`HAProxy` wants the TLS cert is a single .pem file, so we need to combine the chain cert and the domain cert together:

```
cat /etc/letsencrypt/live/example.com/fullchain.pem /etc/letsencrypt/live/example.com/privkey.pem > /etc/haproxy/certs/example.com.pem
```


## Step 5 - Configure HAProxy for SSL Termination

Edit the `/etc/haproxy/haproxy.cfg` config file with the TLS parameters from the [Mozilla Generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy)

```
global
    # set default parameters to the modern configuration
    
    ssl-default-bind-ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    ssl-default-server-ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256
    ssl-default-server-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
```



## Step 6 - Configure HAProxy frontend

Edit `/etc/haproxy/haproxy.cfg` to bind to port 443, and use the cert you obtained for TLS termination

```
frontend www-https
        bind *:443 ssl crt /etc/haproxy/certs/example.com.pem
        reqadd X-Forwarded-Proto:\ https
        http-request set-header X-SSL %[ssl_fc]
        acl letsencrypt-acl path_beg /.well-known/acme-challenge/
        use_backend letsencrypt-backend if letsencrypt-acl
        default_backend www-backend
```

With this configuration, any request that begins with `/.well-known/acme-challenge/` will be sent to `certbot`. This is how we will renew our cert.



## Step 7 - Configure HAProxy backends

Edit `/etc/haproxy/haproxy.cfg`. Our default backend will just forward requests to port 8000. The `letsencrypt-backend` will forward requests to port 9000:

```
backend www-backend
        server www-1 127.0.0.1:8000 check

backend letsencrypt-backend
        # Lets encrypt backend server
        server letsencrypt 127.0.0.1:9000
```
