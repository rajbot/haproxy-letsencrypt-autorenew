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

## Step 4 - Configure HAProxy for SSL Termination

