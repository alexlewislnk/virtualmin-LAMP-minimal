# Virtualmin's Minimal LAMP Server
General instructions for install and setup. The minimal installation excludes the full mail processing stack (IMAP/POP servers, SpamAssassin and ClamAV).

This assumes you have already completed the basic setup of your Ubuntu 20.04 LTS Server. 
I suggest checking out my scripts for [Ubuntu Setup](https://github.com/alexlewislnk/Ubuntu-Setup) and [Emerging Threats Firewall](https://github.com/alexlewislnk/ET-Firewalld).

## Virtualmin Installation

**Add repository for latest version of Apache**
```
add-apt-repository ppa:ondrej/apache2 && apt update -y  
```

**Add repository for latest version of PHP**
```
add-apt-repository ppa:ondrej/php && apt update -y  
```

**Download the Virtualmin installation script**
```
cd /tmp
wget http://software.virtualmin.com/gpl/scripts/install.sh
chmod +rx /tmp/install.sh
```

**Install Virtualmin**
```
/tmp/install.sh --minimal
```

**Make sure all packages are up-to-date**
```
export DEBIAN_FRONTEND=noninteractive
apt update && apt -y full-upgrade
```

**Install Fail2Ban and enable in Virtualmin**
```
apt -y install fail2ban && virtualmin config-system --include Fail2banFirewalld
```

**Make sure Fail2Ban is enabled and running**
```
systemctl enable fail2ban ; systemctl restart fail2ban
```

## Apache and PHP Modifications
**PHP versions and modules**

Install php and common modules for the current supported versions (8.0, 8.1, 8.2) and popular legacy version 7.4.
```
apt -y install php{7.4,8.0,8.1,8.2}-{bcmath,bz2,cgi,cli,common,curl,fpm,gd,igbinary,imagick,intl,mbstring,memcached,mysql,opcache,readline,redis,xml,zip} 
```
Ensure all other PHP legacy versions are removed.
```
apt -y purge php5.6* php7.0* php7.1* php7.2* php7.3*
```
Enable the PHP modules.
```
phpenmod bcmath bz2 curl gd igbinary imagick mbstring memcached opcache readline redis xml zip
```

**HTTP request and response headers**
```
cat > /etc/apache2/conf-available/security.conf <<EOF
ServerTokens Prod
ServerSignature Off
TraceEnable Off
Header unset ETag
FileETag None
Header set X-XSS-Protection "1; mode=block"
Header set X-Content-Type-Options nosniff
EOF
a2enmod headers && systemctl reload apache2
```

**Define browser caching**
```
cat > /etc/apache2/mods-available/expires.conf <<EOF
<IfModule mod_expires.c>
ExpiresActive On
ExpiresDefault "access plus 1 week"
</IfModule>
EOF
a2enmod expires && systemctl reload apache2
```

**Enable http compression**
```
cat > /etc/apache2/mods-available/deflate.conf <<EOF
<IfModule mod_deflate.c>
<IfModule mod_filter.c>
AddOutputFilterByType DEFLATE application/javascript
AddOutputFilterByType DEFLATE application/rss+xml
AddOutputFilterByType DEFLATE application/vnd.ms-fontobject
AddOutputFilterByType DEFLATE application/x-font
AddOutputFilterByType DEFLATE application/x-font-opentype
AddOutputFilterByType DEFLATE application/x-font-otf
AddOutputFilterByType DEFLATE application/x-font-truetype
AddOutputFilterByType DEFLATE application/x-font-ttf
AddOutputFilterByType DEFLATE application/x-javascript
AddOutputFilterByType DEFLATE application/xhtml+xml
AddOutputFilterByType DEFLATE application/xml
AddOutputFilterByType DEFLATE font/opentype
AddOutputFilterByType DEFLATE font/otf
AddOutputFilterByType DEFLATE font/ttf
AddOutputFilterByType DEFLATE image/svg+xml
AddOutputFilterByType DEFLATE image/x-icon
AddOutputFilterByType DEFLATE text/css
AddOutputFilterByType DEFLATE text/html
AddOutputFilterByType DEFLATE text/javascript
AddOutputFilterByType DEFLATE text/plain
AddOutputFilterByType DEFLATE text/xml
</IfModule>
</IfModule>
EOF
a2enmod deflate && systemctl reload apache2
```

**Harden SSL**

Create Diffie-Hellman key pairs
```
openssl dhparam -out /etc/ssl/dhparam.pem 2048
```

Create new Apache SSL config file
```
cp /etc/apache2/mods-available/ssl.conf /etc/apache2/mods-available/ssl.conf.save
cat > /etc/apache2/mods-available/ssl.conf <<EOF
<IfModule mod_ssl.c>
SSLRandomSeed startup builtin
SSLRandomSeed startup file:/dev/urandom 512
SSLRandomSeed connect builtin
SSLRandomSeed connect file:/dev/urandom 512
AddType application/x-x509-ca-cert .crt
AddType application/x-pkcs7-crl .crl
SSLPassPhraseDialog exec:/usr/share/apache2/ask-for-passphrase
SSLSessionCache         shmcb:\${APACHE_RUN_DIR}/ssl_scache(512000)
SSLSessionCacheTimeout  300
SSLProtocol -All +TLSv1.2 +TLSv1.3
SSLCipherSuite ECDH+CHACHA20:ECDH+AESGCM+AES128:ECDH+AESGCM:ECDH+AES128:ECDH+AES256:!aNULL:!SHA1:!AESCCM
SSLCipherSuite TLSv1.3 TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384
SSLHonorCipherOrder On
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
</IfModule>
EOF
sed -i '/SSLProtocol/D' /etc/apache2/apache2.conf && sed -i '/SSLCipherSuite/D' /etc/apache2/apache2.conf
systemctl restart apache2
```

**Define bots and crawlers to block in Apache**
```
mkdir -p /etc/apache2/misc
cat > /etc/apache2/misc/badbots.conf <<EOF
    RewriteEngine On
    RewriteCond %{HTTP_USER_AGENT} ^.*ahrefsbot.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*aspiegelbot.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*baiduspider.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*blexbot.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*cloudfind.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*dotbot.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*go-http-client.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*mauibot.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*mj12bot.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*petalbot.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*project25499.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*proximic.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*pubmatic.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*qihu\ 360.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*screaming\\ frog.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*semrushbot.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*sogou.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*yandex.* [NC,OR]
    RewriteCond %{HTTP_USER_AGENT} ^.*yisouspider.* [NC]
    RewriteRule . - [R=406,L]
EOF
```

## Virtualmin Post-Installation Wizard
From a web browser, log in to the Virtualmin console at port 10000, using the root user credentials, and complete the Post-Installation Wizard. For the initial setup, you should use the server's IP address in the URL instead of FQDN (https://x.x.x.x:10000)

During the setup wizard:
- Preload Virtualmin libraries?  **No**
- Run MySQL database server?  **Yes**
- Run PostgreSQL database server?  **No**
- Set MySQL password  **(Enter the MySQL root password created earlier in this guide)**
- MySQL configuration size  **Leave default settings**
- DNS Configuration  **Check box for Skip check for resolvability**
- Setup default virtual server?  **No**
- Enable SSL on default server?  **No**

At the end of the Wizard, select the option to **Manage Enabled Features and Plugins**. These are the only features that should be enabled:
- Apache website
- Apache SSL website
- MySQL database
- Log file rotation
- Protected web directories

Select **System Settings** on the left menu, then click on **Server Templates**. Click on **Default Settings**
- Website for domain
  - Directive and settings for new websites (add the following line to bottom)
  ```
  IncludeOptional misc/*.conf
  ```
  - Enable HTTP2 protocol for new websites **Yes**
  - Redirect HTTP to HTTPS by default?  **Yes**
- MySQL database
  - MySQL password for new domains  **Generate Randomly**
- PHP options
  - Default PHP exection mode  **FPM**
  - Default PHP version  **8.1.x**

Select **System Settings** on the left menu, then click on **Virtualmin Configuration**
- Defaults for new domains
  - Length of randomly generated password  **16**

Select **System Settings** on the left menu, then click on **Re-check Configuration**

## Final Tweaks

**Set MariaDB/Mysql logging to 10 days**
```
if [ -f /etc/mysql/mysql.conf.d/mysqld.cnf ]; then
cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.backup
sed -i '/binlog_expire_logs_seconds/D' /etc/mysql/mysql.conf.d/mysqld.cnf
cat >> /etc/mysql/mysql.conf.d/mysqld.cnf <<EOF
binlog_expire_logs_seconds = 864000
EOF
fi
if [ -f /etc/mysql/mariadb.conf.d/50-server.cnf ]; then
cp /etc/mysql/mariadb.conf.d/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf.backup
sed -i '/binlog_expire_logs_seconds/D' /etc/mysql/mariadb.conf.d/50-server.cnf
cat >> /etc/mysql/mariadb.conf.d/50-server.cnf <<EOF
binlog_expire_logs_seconds = 864000
EOF
fi
```

**Disable unnecessary services**

If you plan to host DNS elsewhere, disable Bind DNS
```
systemctl mask bind9
```

If you plan to host email elsewhere, disable Dovecot Mail Server
```
systemctl mask dovecot
```

Disable Proftp server (I strongly encourage the use of ssh-based sftp instead of ftp/ftps)
```
systemctl mask proftpd
```

**Harden email encryption**

Create default self-signed cert for Postfix
```
touch ~/.rnd
openssl req -new -x509 -nodes -out /etc/ssl/postfix.pem -keyout /etc/ssl/postfix.key -days 3650 -subj "/C=US/O=$HOSTNAME/OU=Email/CN=$HOSTNAME"

```

Configure Postfix SSL/TLS
```
postconf -e tls_medium_cipherlist=ECDH+AESGCM+AES128:ECDH+AESGCM:ECDH+CHACHA20:ECDH+AES128:ECDH+AES:DHE+AES128:DHE+AES:RSA+AESGCM+AES128:RSA+AESGCM:\!aNULL:\!SHA1:\!DSS
postconf -e tls_preempt_cipherlist=yes
postconf -e smtpd_use_tls=yes
postconf -e smtpd_tls_loglevel=1
postconf -e smtpd_tls_security_level=may
postconf -e smtpd_tls_auth_only=yes
postconf -e smtpd_tls_protocols=\!SSLv2,\!SSLv3,\!TLSv1,\!TLSv1.1
postconf -e smtpd_tls_ciphers=medium
postconf -e smtpd_tls_mandatory_protocols=\!SSLv2,\!SSLv3,\!TLSv1,\!TLSv1.1
postconf -e smtpd_tls_mandatory_ciphers=medium
postconf -e smtpd_tls_cert_file=/etc/ssl/postfix.pem
postconf -e smtpd_tls_key_file=/etc/ssl/postfix.key
postconf -e smtpd_tls_dh1024_param_file=/etc/ssl/dhparam.pem
postconf -e smtp_use_tls=yes
postconf -e smtp_tls_loglevel=1
postconf -e smtp_tls_security_level=may
postconf -e smtp_tls_protocols=\!SSLv2,\!SSLv3,\!TLSv1,\!TLSv1.1
postconf -e smtp_tls_ciphers=medium
postconf -e smtp_tls_mandatory_protocols=\!SSLv2,\!SSLv3,\!TLSv1,\!TLSv1.1
postconf -e smtp_tls_mandatory_ciphers=medium
postconf -e smtp_tls_cert_file=/etc/ssl/postfix.pem
postconf -e smtp_tls_key_file=/etc/ssl/postfix.key
systemctl restart postfix
```

Restrict inbound email

Since we are not using the Virtualmin’s mail services, then let’s lock down the Postfix SMTP server so it cannot be an attack target. We cannot disable it completely as it will be needed to send outbound email from your server. We configure it so connections are only accepted from the server itself.
```
postconf -e inet_interfaces=127.0.0.1
systemctl restart postfix
```

## Setup Default Apache Site to Block IP URL Requests

```
a2dissite 000-default
openssl req -new -x509 -nodes -out /etc/ssl/snakeoil.pem -keyout /etc/ssl/snakeoil.key -days 3650 -subj '/CN=*'
VirtualHost1=""
VirtualHost2=""
ServerName="ServerName"
ServerAlias="ServerAlias"
AliasFlag=0
for i in $(hostname -I)
do
VirtualHost1="$VirtualHost1 $i:80"
VirtualHost2="$VirtualHost2 $i:443"
if [ $AliasFlag = 0 ] ; then
ServerName="$ServerName $i"
AliasFlag=1
else
ServerAlias="$ServerAlias $i"
AliasFlag=2
fi
done
if [ $AliasFlag = 1 ] ; then
ServerAlias=""
fi
cat > /etc/apache2/sites-available/000-default.conf << EOF
<VirtualHost $VirtualHost1>
$ServerName
$ServerAlias
DocumentRoot /var/www/html/
RedirectMatch 400 /(.*)\$
ErrorLog /var/log/apache2/default_error_log
CustomLog /var/log/apache2/default_access_log combined
</VirtualHost>
<VirtualHost $VirtualHost2>
$ServerName
$ServerAlias
DocumentRoot /var/www/html/
RedirectMatch 400 /(.*)\$
ErrorLog /var/log/apache2/default_error_log
CustomLog /var/log/apache2/default_access_log combined
SSLEngine on
SSLCertificateFile /etc/ssl/snakeoil.pem
SSLCertificateKeyFile /etc/ssl/snakeoil.key
SSLCACertificateFile /etc/ssl/snakeoil.pem
</VirtualHost>
EOF
a2ensite 000-default && systemctl reload apache2
```

## Finished – Reboot
This concludes the initial setup and configuration of you Virtualmin LAMP Server. Before creating your virtual webservers, reboot your server to make sure everything starts up correctly.
```
reboot
```

## After Creating a Virtual Website ##
When creating a virtual website, Virtualmin will include SSL settings that will override the hardened global SSL setting we created earlier. Let's remove any errent SSL settings.
```
sed -i '/SSLProtocol/D' /etc/apache2/sites-available/*
sed -i '/SSLCipherSuite/D' /etc/apache2/sites-available/*
systemctl reload apache2
```
