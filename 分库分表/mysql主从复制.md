# 一、创建主从数据库

## 1.创建主库

编写主库配置文件my.cnf

```shell
# Copyright (c) 2017, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301 USA

#
# The MySQL  Server configuration file.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
# Disabling symbolic-links is recommended to prevent assorted security risks
# symbolic-links=0
log-bin=mysql-bin
server-id = 10

sql_mode=STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION

# Custom config should go here
!includedir /etc/mysql/conf.d/
```

编写docker-compose.yml

```yaml
version: '3.1'
services:
  db:
    image: mysql
    container_name: mysql8_master
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 123456
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3339:3306
    volumes:
      - ./data:/var/lib/mysql
      - ./my.cnf:/etc/my.cnf 
```

## 创建从库

编写主库配置文件my.cnf

```shell
# Copyright (c) 2017, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301 USA

#
# The MySQL  Server configuration file.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
# Disabling symbolic-links is recommended to prevent assorted security risks
# symbolic-links=0
log-bin=mysql-bin
server-id = 2

sql_mode=STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION

# Custom config should go here
!includedir /etc/mysql/conf.d/
```

编写docker-compose.yml

```yaml
version: '3.1'
services:
  db:
    image: mysql
    container_name: mysql8_master
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 123456
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3340:3306
    volumes:
      - ./data:/var/lib/mysql
      - ./my.cnf:/etc/my.cnf 
```



注意事项：

- server-id是唯一的，主从不能相同，server-id为1表示mysq数据库为主数据库，server-id为2表示mysql的数据库为从数据库
- 为何必须挂载到/etc/my.cnf 文件上，而不是/etc/mysql/my.cnf 文件上？因为mysql默认配置文件位置在/etc/mysql/my.cnf，挂在方式无法改变容器中文件内容，my.conf内容不会改变，my.cnf中没有我们自定义的配置内容，启动mysql容器会报错

- 在关闭数据库的命令发现mysql关不了，提示Warning: World-writable config file '/etc/my.cnf' is ignored ，大概意思是权限全局可写，任何一个用户都可以写。mysql担心这种文件被其他用户恶意修改，所以忽略掉这个配置文件。这样mysql无法关闭。

下面看下整个过程

**重启MySQL**

```shell

[root@ttlsa ~]# service mysqld stopWarning: World-writable config file '/etc/my.cnf' is ignoredWarning: World-writable config file '/etc/my.cnf' is ignoredMySQL manager or server PID file could not be found![FAILED]

```

可以看到mysql停止不了

**查看my.cnf的权限**

```shell
[root@ttlsa ~]# ls -l /etc/my.cnf-rwxrwxrwx 1 root root 4878 Jul 30 11:31 /etc/my.cnf
```

权限777，任何一个用户都可以改my.cnf，存在很大的安全隐患.

**修复MySQL问题**

```shell
[root@ttlsa ~]# chmod 644 /etc/my.cnf
```

my.cnf设置为用户可读写，其他用户不可写.



# 二、主库中创建同步数据的用户

```shell
#在主库上创建同步用户并授权
CREATE USER 'replicate'@'112.74.41.236' IDENTIFIED BY '123456';
GRANT REPLICATION SLAVE ON *.* TO 'replicate'@'112.74.41.236';
FLUSH PRIVILEGES;
```

**注意**：必须是地址，不能是localhost之类的

# 三、配置从库

## 1.查询主库相关信息

```sql
show master status
```

![image-20200803201451847](G:\soft\Typora\img\image-20200803201451847.png)



## 2.备库连接到主库

```sql
STOP SLAVE; -- 先暂停同步

CHANGE MASTER TO master_host = '112.74.41.236',
master_port = 3339,
master_user = 'replicate',
master_password = '123456',
master_log_file = 'mysql-bin.000003',
master_log_pos = 3230;

START SLAVE; -- 开启同步
```

**说明**

- master_port：Master的端口号，指的是容器的端口号 
- master_user：用于数据同步的用户 
- master_password：用于同步的用户的密码 
- master_log_file：指定 Slave 从哪个日志文件开始复制数据，即上文中提到的 File 字段的值 
- master_log_pos：从哪个 Position 开始读，即上文中提到的 Position 字段的值 
- master_connect_retry：如果连接失败，重试的时间间隔，单位是秒，默认是60秒



## 3.查看是否同步成功

```sql
show slave status;
```

查询slave的状态，Slave_IO_Running及Slave_SQL_Running进程必须正常运行，即YES状态

![image-20200803202236527](G:\soft\Typora\img\image-20200803202236527.png)

![image-20200803202302290](G:\soft\Typora\img\image-20200803202302290.png)

# 四.测试



在主库中创建test数据库并在库中创建user表

![image-20200803202438540](G:\soft\Typora\img\image-20200803202438540.png)



可以发现备库也同步创建了

![image-20200803202541118](G:\soft\Typora\img\image-20200803202541118.png)