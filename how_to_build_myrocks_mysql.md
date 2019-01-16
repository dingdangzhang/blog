####编译安装myrocks mysql

#####一、需要在系统上安装一些依赖的包以及相关工具：

```js
1、sudo yum install cmake gcc-c++ bzip2-devel libaio-devel bison zlib-devel snappy-devel boost-devel
2、sudo yum install gflags-devel readline-devel ncurses-devel openssl-devel lz4-devel gdb git
3、sudo yum install libzstd-devel
4、sudo yum install perl-Digest-MD5 perl-DBD-MySQL perl-DBI MySQL-python 
```

#####二、clone myrocks代码

```js
1、git clone https://github.com/facebook/mysql-5.6.git
2、cd mysql-5.6
3、git submodule init
4、git submodule update
5、cmake . -DCMAKE_BUILD_TYPE=Debug -DWITH_SSL=system -DWITH_ZLIB=bundled -DMYSQL_MAINTAINER_MODE=1 -DENABLE_DTRACE=0 -DWITH_ZSTD=/usr
6、make -j128
7、安装：make install
```
#####三、配置运行
- 1、创建一个配置文件my.cnf

```js
#cat my.cnf
[client]
default-character-set=utf8

[mysqld]
character_set_server=utf8
basedir=/usr/local/mysql    #这里指定mysql安装的根目录，也就是make install的目录
datadir=/path/to/data_base #这里指定数据文件目录
socket=/tmp/mysql.sock    #这里很重要，需要指定成这个目录 否则启动mysql时候出错。
port=3306
server_id=1
user=mysql
default_authentication_plugin=mysql_native_password

rocksdb
default-storage-engine=rocksdb
skip-innodb
default-tmp-storage-engine=MyISAM
collation-server=utf8_bin

log-bin
binlog-format=ROW
```

- 2、安装数据库

```js
 ./mysql_install_db --user=mysql --defaults-file=/path/to/my.cnf --basedir=/usr/local/mysql
```

- 3、启动数据库

```js
./mysqld_safe --defaults-file=/path/to/my.cnf &
```
- 4、查看数据库启动成功没有

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/show_mysql.png)

```js
#ps aux |grep myqql
```


- 5、登陆mysql

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/start_mysql.tiff)

```js
cd /usr/local/mysql/bin
./mysql 
```

