
*Updated 22 March 2024 · Toulouse*

[aw]: https://github.com/svijalove/server/tree/master/documentation/Apache%20%26%20Wordpress

![Svija: SVG-based websites built in Adobe Illustrator][logo]

[logo]: http://files.svija.love/github/readme-logo.png "Svija: SVG-based websites built in Adobe Illustrator"

# Setting up a Server for Sleeping Sites

[original instructions](https://gist.github.com/talyguryn/bd0f30ab3eb183afbe9521261adfbc60)

The goal is to be able to direct any domain name to this server, and it will automatically serve up the sleeping page.

To do this, we use a "wildcard" certificate that will work with any domain name.

To request it, put the domain into a variable:
```
DOMAIN=svija.site
```
Request the certificate:
```
certbot certonly --manual -d *.$DOMAIN -d $DOMAIN \
--agree-tos \
--manual-public-ip-logging-ok \
--preferred-challenges dns-01 \
--server https://acme-v02.api.letsencrypt.org/directory \
--register-unsafely-without-email \
--rsa-key-size 4096
```
The server will respond:
```
Please deploy a DNS TXT record under the name:

_acme-challenge.svija.site.

with the following value:

p8l6h2k1rhb_VvOSfYlhxpFaX7P66D3ikhy8eyMM5N0
```
Add a TXT record with the requested string via the domains tab at [cloud.linode.com](https://cloud.linode.com/domains).

Next, check if the TXT record has been deployed (in a separate window):
```
host -t txt _acme-challenge.svija.site
```
Press ENTER to continue. The server will respond with:
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/svija.site/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/svija.site/privkey.pem
This certificate expires on 2024-06-20.
These files will be updated when the certificate renews.
```
<details><summary>Seemingly unnecessary bit</summary>

```
cd /etc/nginx/snippets/
vi ssl.conf
```
paste the following:
```
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets on;

ssl_protocols TLSv1.2;
ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES256-SHA384;
ssl_ecdh_curve secp384r1;
ssl_prefer_server_ciphers on;

ssl_stapling on;
ssl_stapling_verify on;

add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; preload";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
```

</details>

Create the `certs` directory:
```
cd /etc/nginx/snippets/
mkdir certs
cd certs
```
Create the cerfificate list:
```
vi svija.site
```
Paste the following text:
```
ssl_certificate /etc/letsencrypt/live/svija.site/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/svija.site/privkey.pem;
ssl_trusted_certificate /etc/letsencrypt/live/svija.site/fullchain.pem;
```
Create the NginX config file:
```
cd /etc/nginx/sites-available/
vi svija.site
```
Paste the following contents:
```
server {
    listen 80;
    listen 443 ssl;

    # Listen abc.svija.site and *.abc.svija.site domains
    # Get $subdomain variable from domain name
    server_name ~^(?<subdomain>.+)\.svija\.site svija.site;

    # Include ssl configs and cert's params
    include snippets/certs/svija.site;

    # Return the "sleeping site" page
    root /opt/sleeping/html;
    index index.html;
}
```
Enable the site:
```
ln -s /etc/nginx/sites-available/svija.site /etc/nginx/sites-enabled/svija.site
service nginx restart
```
