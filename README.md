# percona

#Below are the steps to installing and configuring the Percona XtraDB Cluster

	Add the percona xtradb cluster repo
o	yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm

	Enable the XtraDB Cluster repo:
o	percona-release setup pxc-57

	Enable the XtraDB tools:
o	sudo percona-release enable tools release

	Install the Percona XtraDB Cluster packages:
o	dnf install Percona-XtraDB-Cluster-57

	Add these ports on firewalld :
o	 firewall-cmd --permanent --add-port=3306/tcp
o	firewall-cmd --permanent --add-port=4567/tcp
o	firewall-cmd --permanent --add-port=4567/udp
o	firewall-cmd --permanent --add-port=4568/tcp
o	firewall-cmd --permanent --add-port=4444/tcp
o	firewall-cmd --reload

	Disable SeLinux
o	setenforce 0

	Add the following configuration variables to /etc/percona-xtradb-cluster.conf.d/wsrep.cnf on the first node

[mysqld] wsrep_provider=/usr/lib64/galera3/libgalera_smm.so 
wsrep_cluster_name=pxc-cluster wsrep_cluster_address=gcomm://192.168.102.31,192.168.102.32,192.168.102.33 
wsrep_node_name=pxc1 
wsrep_node_address=192.168.102.31 
wsrep_sst_method=xtrabackup-v2 
wsrep_sst_auth=sstuser:passw0rd 
pxc_strict_mode=DISABLE 
binlog_format=ROW 
default_storage_engine=InnoDB 
innodb_autoinc_lock_mode=2

	Start the first node with the following command
o	systemctl start mysql@bootstrap.service

	Get temporary password of root
o	grep 'A temporary password' /var/log/mysqld.log |tail -1

	Make MySQL secure installation
o	mysql_secure_installation

	login to the fisrt node mysql
o	mysql -u root -p

	To make sure that the cluster has been initialized, run the following:
o	show status like 'wsrep%';

	Before adding other nodes to your new cluster, create a user for SST and provide the necessary privileges for that user account. The credentials must match those specified when Configuring Nodes for Write-Set Replication.
o	CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'passw0rd';
o	GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
o	FLUSH PRIVILEGES;

	Starting the Second Node
o	systemctl start mysqld

	To check the status of the second node, run the following:
o	show status like 'wsrep%';

	Starting the Third Node
o	systemctl start mysqld

	To check the status of the Third node, run the following:
o	show status like 'wsrep%';

#Below are the steps to installing and configuring ProxySQL

01). Create the ProxySQL YUM Repository

cat > /etc/yum.repos.d/proxysql.repo << EOF
[proxysql]
name=ProxySQL YUM repository
baseurl=https://repo.proxysql.com/ProxySQL/proxysql-2.7.x/centos/\$releasever
gpgcheck=1
gpgkey=https://repo.proxysql.com/ProxySQL/proxysql-2.7.x/repo_pub_key
EOF

02). Install ProxySQL

sudo yum clean all
sudo yum install proxysql -y

03). Configure MySQL interface to use standard port

sudo sed -i 's/^mysql-interfaces=.*/mysql-interfaces=0.0.0.0:3306/' /etc/proxysql.cnf


04). Start and Enable ProxySQL

sudo systemctl start proxysql
sudo systemctl enable proxysql

05). Verify Installation

proxysql --version
sudo systemctl status proxysql

06). Log in to ProxySQL Admin Interface

mysql -u admin -padmin -h 127.0.0.1 -P6032

07). Cluster Node Configuration

INSERT INTO mysql_servers (hostname,port,hostgroup_id) VALUES
('192.168.10.31',3306,10),
('192.168.10.32',3306,11),
('192.168.10.33',3306,11);

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;

08). Query Routing Rules

INSERT INTO mysql_query_rules (rule_id,active,match_pattern,destination_hostgroup,apply) VALUES
(1,1,'^SELECT.*FOR UPDATE$',10,1),
(2,1,'^SELECT',11,1),
(3,1,'.*',10,1);

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;

09). Application User Setup

INSERT INTO mysql_users (username,password,default_hostgroup) VALUES
('user','Password',10);

LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;

10). Monitoring Configuration

UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_username';
UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_password';

LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;


UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_username';
UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_password';
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;


11). Failover Handling with Scheduler

DELETE FROM scheduler WHERE id=1; 

INSERT INTO scheduler (id,active,interval_ms,filename,arg1) VALUES
(1,1,5000,'/usr/bin/proxysql_galera_checker','-h=192.168.102.31,192.168.102.32,192.168.102.33 -u=monitor -p=monitor_pass --writer-hg=10 --reader-hg=11 --mode=singlewrite');

INSERT INTO scheduler (id, interval_ms, filename, arg1) VALUES
(1, 5000, '/usr/bin/proxysql_galera_checker', '-h=10.0.1.80,10.0.1.81,10.0.1.82 -u=monitor -p=monitor --writer-hg=10 --reader-hg=11 --mode=singlewrite');


LOAD SCHEDULER TO RUNTIME;
SAVE SCHEDULER TO DISK;


CREATE USER 'user'@'%' IDENTIFIED BY 'Password';
GRANT ALL PRIVILEGES ON *.* TO 'db'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';
GRANT USAGE, REPLICATION CLIENT ON *.* TO 'monitor'@'%';
FLUSH PRIVILEGES;


SHOW VARIABLES LIKE '%timeout%';
SELECT user,host FROM mysql.user WHERE user IN ('monitor','doxmate');
SELECT * FROM runtime_mysql_servers;



On all node check most sequence num 
#cat /data/mysql/grastate.dat

If most sequence num node change 
#vi /data/mysql/grastate.dat 
safe_to_bootstrap: 0 >>>> safe_to_bootstrap: 1

Bootstrap the Cluster on That Node
systemctl start mysql@bootstrap.service

Other node start nomally
systemctl start mysqld




