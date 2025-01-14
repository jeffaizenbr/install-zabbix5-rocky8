# install-zabbix5-rocky8

#ORIGIN = brppzbm01 <br>
#DESTINY = brpldsinfzbs01

user postgres crontab
```bash
# Daily base backup of postgres databases
0 22 * * *  /var/lib/pgsql/pg_databasebackup.sh >> "/tmp/$(hostname).$(date +\%Y\%m\%d).pg_databasebackup.log" 2>&1
```
user root crontab
```bash
#Ansible: Rogier's DSG Report
5 * * * * /usr/local/report/dsg_linux_system_inventory.pl > /var/log/report/dsg_linux_system_inventory.log 2>&1
#
57 * * * * /data/dsg_linux_system_report/bin/refresh_cmdb.sh                                 > /data/dsg_linux_system_inventory/log/refresh_cmdb.log 2>&1
```


>> /var/lib/pgsql/pg_databasebackup.sh <<
```bash
#!/bin/bash

PORT_NUMBER=5432
BACKUP_DIR="/mnt/backup/OnDisk/brppzbm01/pgbackup"
EXITCODE=0

function pg_databasebackup ()
{

        #outpu start time
        echo "##############################################################################################################################"
        echo "START: $(date)"
        echo

        #get list of databases to backup
        DATABASES=$(psql -p $PORT_NUMBER -At -c "select datname from pg_database where not datistemplate and datallowconn order by datname;" postgres)

        #if no databases caputred then quit
        if [[ -z ${DATABASES} ]]
        then
                #error getting list of databses, set exit code and provide message
                EXITCODE=1
                echo "ERROR: No databases found, confirm port number: ${PORT_NUMBER} is correct and that cluster is online and accessible!!!"
        else
                #for each database
                for DATABASE in ${DATABASES}
                do
                        #if backup directory does not exist create it
                        if [ ! -d "${BACKUP_DIR}/${DATABASE}" ]; then mkdir "${BACKUP_DIR}/${DATABASE}"; fi

                        #set backup filename
                        BACKUPFILENAME="${BACKUP_DIR}/${DATABASE}/pg_dump.$(hostname).${PORT_NUMBER}.${DATABASE}.$(date +\%Y\%m\%d_\%H\%M\%S).custom"

                        #output backup command used
                        echo pg_dump -Fc -p ${PORT_NUMBER} "${DATABASE}" -f "${BACKUPFILENAME}"

                        #backup database
                        if ! pg_dump -Fc -p ${PORT_NUMBER} "${DATABASE}" -f "${BACKUPFILENAME}"
                        then
                                #error backing up, set exit code and provide message
                                EXITCODE=1
                                echo "ERROR: Backup operation for database ${DATABASE} unsuccessful!!!"
                        else
                                echo "INFO : Backup of database ${DATABASE} completed."

                                #delete old backup
                                #echo "INFO : Deleting following backups files older than ${DAYS_TO_RETAIN} days:"
                                #find "${BACKUP_DIR}/${DATABASE}" -maxdepth 1 -name "pg_dump.$(hostname).${PORT_NUMBER}.${DATABASE}.*.custom" -mtime +${DAYS_TO_RE$
                                #find "${BACKUP_DIR}/${DATABASE}" -maxdepth 1 -name "pg_dump.$(hostname).${PORT_NUMBER}.${DATABASE}.*.custom" -mtime +${DAYS_TO_RE$

                        fi
                        echo
                done
        fi


        #provide completion message
        if [ ${EXITCODE} -eq 0 ]; then
                echo "END  : $(date), SUCCESS"
        else
                echo "END  : $(date), ERRORS ENCOUNTERED!!! CHECK PREVIOUS MESSAGES!!!"
        fi

        exit ${EXITCODE}
}

pg_databasebackup;
```
```bash
```

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
vim /etc/php-fpm.d/zabbix.conf
php_value[date.timezone] = America/Toronto
```
## add database access on zabbix server conf 
```bash
vim /etc/zabbix/zabbix_server.conf
```

start all services
```bash
systemctl enable --now zabbix-server zabbix-agent httpd php-fpm && tail -f /var/log/zabbix/zabbix_server.log
```




# Install zabbix proxy


## add repo
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/8/x86_64/zabbix-release-latest-5.0.el8.noarch.rpm
dnf clean all
```
## install zabbix
```bash
dnf install zabbix-proxy-pgsql
```
## install postgresql 13
```bash
dnf module list postgresql
sudo dnf module enable postgresql:16
sudo dnf install postgresql-server
sudo postgresql-setup --initdb
sudo systemctl enable --now  postgresql
```
## add user and database
```bash
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix_proxy
```
## insert zabbix data into database
```bash
zcat  /usr/share/doc/zabbix-proxy-pgsql/schema.sql.gz | sudo -u zabbix psql zabbix_proxy
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
## edit zabbix_proxy configuration file and modify "Server=, Hostname=, database password"
```bash
vim /etc/zabbix/zabbix_proxy.conf
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

# Process do dump and restore
```bash
pg_dump -U postgres -F c -b -v -f /tmp/zabbix_dump.dump zabbix

psql -U postgres -d zabbix -c "DROP SCHEMA public CASCADE;"
psql -U postgres -c "DROP DATABASE IF EXISTS zabbix;"
psql -U postgres -c "CREATE DATABASE zabbix;"

pg_restore -U postgres -d zabbix -v /tmp/zabbix_dump.dump
```

## DEFINITIVE DUMP AND RESTORE

#NOTES
backups located at > /mnt/backup/OnDisk/brppzbm01/pgbackup/zabbix <br>
copy to local storage > /data/pgsql/DUMP

#DUMP
```bash
pg_dump -U postgres -F c -b -v -f /tmp/zabbix_dump.dump zabbix
```

#RESTORE
```bash
DROP DATABASE zabbix;
CREATE DATABASE zabbix;
GRANT ALL PRIVILEGES ON DATABASE zabbix TO zabbix;
CREATE ROLE report;
pg_restore -U postgres -d zabbix --jobs=4 --verbose pg_dump.brppzbm01.prd.dsg-internal.5432.zabbix.20250112_220001.custom
```


cp -rfp /mnt/backup/OnDisk/brppzbm01/pgbackup/zabbix/pg_dump.brppzbm01.prd.dsg-internal.5432.zabbix.20250112_220001.custom .




