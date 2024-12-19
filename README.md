# install-zabbix5-rocky8

## Install zabbix repo
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/8/x86_64/zabbix-release-latest-5.0.el8.noarch.rpm
```
## install zabbix front-end 
```bash
dnf install zabbix-server-pgsql zabbix-web-pgsql zabbix-apache-conf zabbix-agent
```
## install postgresql 13
```bash
dnf module list postgresql
sudo dnf module enable postgresql:13
sudo dnf install postgresql-server
sudo postgresql-setup --initdb
sudo systemctl enable --now  postgresql
```
## Login to user postgres
```bash
su - postgres
psql
```
## Create zabbix database
```bash
CREATE DATABASE zabbix
    WITH OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'C'
    LC_CTYPE = 'C'
    TEMPLATE = template0;
```
## Create Zabbix user
```bash
CREATE USER zabbix WITH PASSWORD '<PASSWORD>';
```
## Grant privileges do zabbix user
```bash
GRANT ALL PRIVILEGES ON DATABASE zabbix TO zabbix;
```

## execute zcat with zabbix data
```bash
zcat /usr/share/doc/zabbix-server-pgsql/create.sql.gz | sudo -u zabbix psql zabbix
```
## Edit pg_hba file to allow MD5 login
```bash
vim /var/lib/pgsql/data/pg_hba.conf
local   all             all                                     md5
```
## Restart postgresql
```bash
systemctl restart postgresql && systemctl enable postgresql
```
## configure timezone in PHP
```bash
php_value[date.timezone] = America/Toronto
```
start all services
```bash
systemctl enable --now zabbix-server zabbix-agent httpd php-fpm && tail -f /var/log/zabbix/zabbix_server.log
```


# Install zabbix proxy

## Install zabbix repo
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/8/x86_64/zabbix-release-latest-5.0.el8.noarch.rpm
```
## install zabbix proxy
```bash
dnf install zabbix-proxy-sqlite3
```
## Generate TLS PSKI for proxy 
```bash
openssl rand -hex 32 > /etc/zabbix/zabbix_proxy.psk
```
## add TLS PSKI configuration on zabbix file
```bash
TLSConnect=psk
TLSPSKFile=/etc/zabbix/zabbix_proxy.psk
TLSPSKIdentity=zabbix_proxy
```

```bash
```

```bash
```

```bash
```

```bash
```
```bash
```

```bash
```

```bash
```

```bash
```

```bash
```
