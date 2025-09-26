
<!-- vim: set foldmethod=marker fmr=###,--- :-->

*Updated 5 September, 2025*

[aw]: https://github.com/svijasvg/server/tree/master/documentation/Apache%20%26%20Wordpress

![Svija: SVG-based websites built in Adobe Illustrator][logo]

[logo]: http://files.svija.com/github/readme-logo.png "Svija: SVG-based websites built in Adobe Illustrator"

### Svija Server Setup

---

<details><summary>Initial Build</summary>

### Initial Build

- start with blank Debian 13 image at [akamai](https://cloud.linode.com)  
  go to `•••` › `rebuild` to reinitialize an existing server
- use `Shared CPU` / `Nanode 1GB`
- get a password from [files.svija.com/passwords](https://files.svija.com/passwords)
- don't include SSH keys
- uncheck "Disk Encryption"
- requires 1 minute to start up

The following two steps are necessary even when rebuilding an existing server:
```
# on local computer
ssh-copy-id root@000.000.000.000          # local: SSH key
```
```
# correct email address then run on server
ssh-keygen -t rsa -C "EMAIL" -N "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub
```
Paste the key into [Github Settings](https://github.com/settings/keys)
```
# on server
apt update -y
apt-get install -y git
```
### Set up Git and bash profiles at this time.

----------

</details><details><summary>Installation Script</summary>

### Installation Script

```
cd /opt
touch script.sh
chmod 777 script.sh
vi script.sh
```
Paste the following into script.sh:
```
#———————————————————— avoid needing to interact

export DEBIAN_FRONTEND=noninteractive
# export DEBIAN_FRONTEND=dialog # to reset

#———————————————————— fix locale issue at login

# generate the en_US.UTF-8 locale
sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen

unset LC_CTYPE
unset LC_ALL
update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8

#———————————————————— set the time zone

ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
echo "Europe/Paris" > /etc/timezone
dpkg-reconfigure -f noninteractive tzdata

#———————————————————— update the server

apt dist-upgrade -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confnew"
# --force-confdef = automatically use the default answer for prompts
# --force-confnew = new will accept the new maintainer version of the config file

#———————————————————— packages

apt-get install -y ufw
apt-get install -y nginx
apt-get install -y gettext
apt-get install -y uwsgi uwsgi-plugin-python3
apt-get install -y libpq-dev postgresql postgresql-contrib
apt-get install -y virtualenvwrapper
apt-get install -y python3-venv
apt-get install -y rsync
apt-get install -y certbot python3-certbot-nginx

echo "postfix postfix/main_mailer_type string 'Internet Site'" | sudo debconf-set-selections
echo "postfix postfix/mailname string DOMAIN" | sudo debconf-set-selections
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y postfix

apt-get install -y mailutils
apt-get install -y opendkim opendkim-tools
apt-get install -y swaks

#———————————————————— limited user is used by uwsgi

# -d specify directory -m make dir if doesn't exist
useradd -m -d /home_hidden/limited limited

adduser limited sudo
chown -R limited:limited /home

#———————————————————— set up firewall

ufw allow 22/tcp            # ssh
ufw allow 80/tcp            # http
ufw allow 443               # https
ufw allow 873/tcp           # rsync daemon
ufw allow out 25/tcp        # dkim email signing

ufw default deny incoming
ufw default allow outgoing
ufw --force enable

#———————————————————— rsyncd passwords file

touch /etc/rsyncd.scrt

#———————————————————— certbot

certbot register \
  -n \
  --agree-tos \
  --email EMAIL \
  --no-eff-email

# registers account with Certbot/Let’s Encrypt
# --no-eff-email prevents promotional emails
#             -n avoids prompts
# -d is not needed since we’re not yet requesting a certificate

#———————————————————— ftp access
#                     https://linux.die.net/man/5/sshd_config

chown root /home      # make `root` the owner of the home directory
groupadd restricted   # add the new group

#———————————————————— change sendifle to off — we use Django's caching

sudo sed -i 's/sendfile on;/sendfile off;/g' /etc/nginx/nginx.conf

#———————————————————— add defaults to /etc/rsyncd.conf

tee /etc/rsyncd.conf > /dev/null <<'EOF'
log file = /opt/logs/rsyncd.log
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
EOF

#———————————————————— enable Rsync Daemon on Startup

sed -i 's/RSYNC_ENABLE=false/RSYNC_ENABLE=true/g' /etc/default/rsync

#———————————————————— replace line 115 of /etc/ssh/sshd_config

sudo sed -i 's|^Subsystem sftp[[:space:]]\+/usr/lib/openssh/sftp-server|Subsystem sftp internal-sftp|' /etc/ssh/sshd_config

#———————————————————— add at end of file

sudo tee -a /etc/ssh/sshd_config > /dev/null <<'EOF'

Match Group restricted
    ChrootDirectory %h
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
EOF

#———————————————————— enable virtualenv

cat << 'EOF' >> ~/.profile
export WORKON_HOME=/opt/venv
export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3
source /usr/share/virtualenvwrapper/virtualenvwrapper.sh
EOF

source ~/.profile

#———————————————————— new venv

cd /opt/venv
python3 -m venv djangoEnv
workon djangoEnv

#———————————————————— packages

pip install psycopg2-binary
pip install django
pip install django-ckeditor
pip install wheel
pip install django-model-utils
python3 -m pip install requests

```
----------

</details><details><summary>This Repository</summary>

### This Repository (Pick One)

Paste **one of the two** following sections into the `script.sh`:
```
# master branch

#———————————————————— per-site logging

cd /opt && mkdir logs

#———————————————————— clone repo

git clone git@github.com:/svijasvg/server.git
cd server

#———————————————————— automatically remove www from URLs

mv remove-www /etc/nginx/sites-available/
ln -s /etc/nginx/sites-available/remove-www /etc/nginx/sites-enabled

#———————————————————— install uwsgi service

mkdir -p /etc/uwsgi/sites
mv uwsgi.service /etc/systemd/system/

#———————————————————— enable Nginx & uWSGI to start at boot

systemctl enable nginx
systemctl enable uwsgi

#———————————————————— clean up

cd /opt && rm -rf server
rm /etc/nginx/sites-enabled/default

```
```

# beta branch

#———————————————————— per-site logging

cd /opt && mkdir logs

#———————————————————— clone repo

git clone -b beta git@github.com:/svijasvg/server.git
cd server

#———————————————————— automatically remove www from URLs

mv remove-www /etc/nginx/sites-available/
ln -s /etc/nginx/sites-available/remove-www /etc/nginx/sites-enabled

#———————————————————— install uwsgi service

mkdir -p /etc/uwsgi/sites
mv uwsgi.service /etc/systemd/system/

#———————————————————— enable Nginx & uWSGI to start at boot

systemctl enable nginx
systemctl enable uwsgi

#———————————————————— clean up

cd /opt && rm -rf server
rm /etc/nginx/sites-enabled/default

```
----------

</details><details><summary>Install Svija</summary>

### Install Svija (Pick One)

Paste **one of the three** following sections into the `script.sh`:
```
#———————————————————— install Svija Cloud (NOT maintaining)

pip install git+https://github.com/svijasvg/cloud.git@master#egg=django-svija

```
```
#———————————————————— install Svija Cloud Beta (NOT maintaining)

pip install git+https://github.com/svijasvg/cloud.git@beta#egg=django-svija

```
```
#———————————————————— install Svija Cloud Beta (maintaining)

cd /opt
git clone -b beta git@github.com:/svijasvg/cloud.git
cd cloud
pip install -e /opt/cloud

```
----------

</details>

Search/replace the following in `script.sh`:
- 1x EMAIL —› certbot email address (name+servername+certbot@example.com)
- 1x DOMAIN —› domain.tld that will be used by postfix
```
# takes about 4 minutes
source script.sh
```
```
reboot
```
Continue to [email setup](mail.md).

