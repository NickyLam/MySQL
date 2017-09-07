
# MySQL Problems  List

## P1.when starting " service mysql3307 start",it occurs   
[root@bogon mysql3306]# service mysql3306 start
Starting MySQL.170905 08:32:37 mysqld_safe error: log-error set to '/var/log/mariadb/mariadb.log', however file don't exists. Create writable for user 'mysql'.
 ERROR! The server quit without updating PID file (/var/lib/mysql/bogon.pid).

### solution 1:    
check whether /etc/rc.d/init.d/mysql3306 has been modified in the following:
conf=/etc/my.cnf  ->  conf=/etc/my3306.cnf
$bindir/mysqld_safe --datadir="$datadir" --pid-file="$mysqld_pid_file_path" $other_args >/dev/null &
->
$bindir/mysqld_safe --defaults-file=/etc/my3307.cnf --datadir="$datadir" --pid-file="$mysqld_pid_file_path" $other_args >/dev/null 2>&1 &

### solution 2:   





##  P2:cannot connect to the server through "/tmp/mysql.sock"     
[root@test-lingguimao mysql3307]# mysql -uroot
ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)

### solution 1:    
try to log in using: mysql -P3307 -S/tmp/mysql3307.sock -uroot -p123456
3307 is the mysql port, and 123456 is the root password.

### solution 2:    
change the privilege of /tmp to mysql like: chown -R mysql:mysql /tmp


## P3:the master and the slave have the same ids.
### solution 1:     
make sure that the scripts"change master to ..." is executed on the slave server instead of the master server.

### solution 2:
check the server id by:
mysql>show variables like "server_id";
or check the slave and master port by:
mysql>show variables like "%port%";