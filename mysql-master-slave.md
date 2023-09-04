
# master
yum clean all
yum makecache
yum -y install mariadb mariadb-server
touch /var/log/mariadb/mariadb.log
chown mysql.mysql /var/log/mariadb/mariadb.log
echo -e "LimitNOFILE=100000\nLimitMEMLOCK=100000" >> /usr/lib/systemd/system/mariadb.service
systemctl start mariadb.service

mysqladmin -u root password comleader@123
mysql -u root --password=comleader@123
GRANT REPLICATION SLAVE ON *.* TO 'copy'@'%' IDENTIFIED BY 'comleader@123';
FLUSH PRIVILEGES;
EXIT

mkdir -p /data/mysql
mkdir -p /data/logbin/mysql-bin
echo -e "[mysqld]\nserver-id=1\nbinlog-format=row\nlog_bin=/data/logbin/mysql-bin\nlog-basename=master\ncharacter_set_server=utf8\ndefault_storage_engine=InnoDB\ndatadir=/data/mysql\nsocket=/var/lib/mysql/mysql.sock\n[mysqld_safe]\nlog-error=/var/log/mariadb/mariadb.log\npid-file=/var/run/mariadb/mariadb.pid\n`echo \!`includedir /etc/my.cnf.d" > /etc/my.cnf
chown -R mysql:mysql /data/mysql
chown -R mysql:mysql /data/logbin/mysql-bin
sysctl vm.overcommit_memory=1
systemctl enable mariadb.service
systemctl restart mariadb.service

# slave

echo -e "[base]\nname=CentOS- - Base\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/os/\$basearch/\nenabled=1\ngpgcheck=0\n[updates]\nname=CentOS- - Updates\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/updates/\$basearch/\nenabled=1\ngpgcheck=0\n[extras]\nname=CentOS- - Extras\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/extras/\$basearch/\nenabled=1\ngpgcheck=0">/etc/yum.repos.d/CentOS-Base.repo
echo -e "[centos-openstack-stein]\nname=CentOS-7 - OpenStack stein\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/cloud/\$basearch/openstack-stein/\ngpgcheck=0\nenabled=1\nexclude=sip,PyQt4">/etc/yum.repos.d/CentOS-OpenStack-stein.repo
yum clean all
yum makecache
yum -y install mariadb mariadb-server
touch /var/log/mariadb/mariadb.log
chown mysql.mysql /var/log/mariadb/mariadb.log

mkdir -p /data/mysql
echo -e "[mysqld]\nserver-id=2\nread_only=ON\nrelay_log=relay-log\nrelay_log_index=relay-log.index\ndatadir=/data/mysql\nsocket=/var/lib/mysql/mysql.sock\n[myqsld_safe]\nlog-error=/var/log/mariadb/mariadb.log\npid-file=/var/run/mariadb/mariadb.pid\n`echo \!`includedir /etc/my.cnf.d" > /etc/my.cnf
chown -R mysql:mysql /data/mysql
systemctl start mariadb.service
mysqladmin -u root password comleader@123
mysql -u root --password=comleader@123
CHANGE MASTER TO MASTER_HOST="$master_ip",MASTER_USER='copy',MASTER_PASSWORD='comleader@123',MASTER_PORT=3306,MASTER_LOG_FILE='master-bin.000001',MASTER_LOG_POS=0,MASTER_CONNECT_RETRY=10;
START SLAVE;
EXIT


systemctl enable mariadb.service
systemctl restart mariadb.service
