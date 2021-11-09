# Virtualmin's Minimal LAMP Server
General instructions for install and setup. The minimal installation excludes the full mail processing stack (IMAP/POP servers, SpamAssassin and ClamAV).

This assumes you have already completed the basic setup of your Ubuntu 20.04 LTS Server. 
I suggest checking out my scripts for [Ubuntu Setup](https://github.com/alexlewislnk/Ubuntu-Setup) and [Emerging Threats Firewall](https://github.com/alexlewislnk/ET-Firewalld).

## Install Virtualmin

**Add repository for latest version of Nginx**
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

**Run the install script**
```
/tmp/install.sh --minimal
```

**Configure Fail2Ban to work with the Firewall**
```
virtualmin config-system --include Fail2banFirewalld
```

**Make sure Fail2Ban is enabled and running**
```
systemctl enable fail2ban ; systemctl restart fail2ban
```

## Move MySQL Data Directory
I like to locate the MySQL data under the /home directory so everything for Virtualmin is under a single path. This can be helpful if you later need to add disk space my moving /home to a separate drive.

**Stop the MySQL service**
```
service mysql stop
```

**Create new data directory**
```
mkdir /home/mysql
chown mysql:mysql /home/mysql
```

**Define new data directory in MySQL config file**
```
cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.save
```
```
sed -i '/datadir/d' /etc/mysql/mysql.conf.d/mysqld.cnf
```
```
echo "datadir = /home/mysql" >> /etc/mysql/mysql.conf.d/mysqld.cnf
```

**Update AppArmor**
```
echo "alias /var/lib/mysql/ -> /home/mysql/," >> /etc/apparmor.d/tunables/alias
service apparmor restart
```

**Initialize New Data Directory** *(don’t worry, we will add a password during the Virtualmin setup later)*
```
mysqld --initialize-insecure
```

**Start MySQL**
```
service mysql start
```

**Create SQL System Maintenance User** *(This will include a new longer random password for the maintenance user than normally provided by Ubuntu's setup)*
```
RANDOM1=`< /dev/urandom tr -dc '[:alnum:]' | head -c${1:-64}`
cp /etc/mysql/debian.cnf /etc/mysql/debian.cnf.bak
cat > /etc/mysql/debian.cnf <<EOF
# Automatically generated for Debian scripts. DO NOT TOUCH!
[client]
host     = localhost
user     = debian-sys-maint
password = $RANDOM1
socket   = /var/run/mysqld/mysqld.sock
[mysql_upgrade]
host     = localhost
user     = debian-sys-maint
password = $RANDOM1
socket   = /var/run/mysqld/mysqld.sock
basedir  = /usr
EOF
echo "CREATE USER 'debian-sys-maint'@'localhost' IDENTIFIED BY '$RANDOM1';" | mysql -u root
echo "GRANT ALL PRIVILEGES ON *.* TO 'debian-sys-maint'@'localhost';" | mysql -u root
echo "GRANT PROXY ON ''@'' TO 'debian-sys-maint'@'localhost' WITH GRANT OPTION;" | mysql -u root
```

**Create Random Password for MySQL root user**
```
RANDOM1=`< /dev/urandom tr -dc '[:alnum:]' | head -c${1:-32}`
echo "ALTER USER 'root'@'localhost' IDENTIFIED BY '$RANDOM1';" | mysql -u root
echo "
MySQL root password has been set to $RANDOM1
Please record password for use later in this setup 
and save in your password manager."
```

## Nginx and PHP Modifications
**PHP Versions and Modules**

Install the latest PHP version 8 and common modules.
```
apt -y install php8.0 php8.0-{bcmath,bz2,cgi,cli,common,curl,fpm,gd,igbinary,imagick,mbstring,memcached,mysql,opcache,readline,redis,xml,zip}
```
Install the latest PHP version 7 and common modules.
```
apt -y install php7.4 php7.4-{bcmath,bz2,cgi,cli,common,curl,fpm,gd,igbinary,imagick,mbstring,memcached,mysql,opcache,readline,redis,xml,zip} 
```
Remove all other versions of PHP.
```
apt -y purge php5.6* php7.0* php7.1* php7.2* php7.3* php8.1*
```
Enable the PHP modules.
```
phpenmod bcmath bz2 curl gd igbinary imagick mbstring memcached opcache readline redis xml zip
```

**Apache Modifications**

Customization of HTTP request and response headers
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
```

Enable the mod_headers
```
a2enmod headers
```

Instruct browsers to allow cacheable content to be fetched from the browser’s cache for up to a week
```
cat > /etc/apache2/mods-available/expires.conf <<EOF
<IfModule mod_expires.c>
ExpiresActive On
ExpiresDefault "access plus 1 week"
</IfModule>
EOF
```
