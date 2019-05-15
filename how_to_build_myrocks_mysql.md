#### 编译安装myrocks mysql

##### 一、需要在系统上安装一些依赖的包以及相关工具：

```js
1、sudo yum install cmake gcc-c++ bzip2-devel libaio-devel bison zlib-devel snappy-devel boost-devel
2、sudo yum install gflags-devel readline-devel ncurses-devel openssl-devel lz4-devel gdb git
3、sudo yum install libzstd-devel
4、sudo yum install perl-Digest-MD5 perl-DBD-MySQL perl-DBI MySQL-python 
```

##### 二、clone myrocks代码

```js
1、git clone https://github.com/facebook/mysql-5.6.git
2、cd mysql-5.6 #这里获取的分支是fb-mysql-5.6.35
3、git submodule init
4、git submodule update
5、cmake . -DCMAKE_BUILD_TYPE=Debug -DWITH_SSL=system -DWITH_ZLIB=bundled -DMYSQL_MAINTAINER_MODE=1 -DENABLE_DTRACE=0 -DWITH_ZSTD=/usr
6、make -j128
7、安装：make install
```

#此时这里编译可能存在失败

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/myrocksb_complie_error.png)

#需要修改代码进行修复

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/myrocks_fix_complie_error.png)


##### 三、配置运行
- 1、创建一个配置文件my.cnf

```js
#cat my.cnf
[client]
default-character-set=utf8

[mysqld]
character_set_server=utf8
basedir=/usr/local/mysql    #这里指定mysql安装的根目录，也就是make install的目录
datadir=/path/to/data_base #这里指定数据文件目录
socket=/tmp/mysql.sock    #这里很重要，需要指定成这个目录,没有在手动创建一个目录 否则启动mysql时候出错。
port=3306
server_id=1
user=mysql
default_authentication_plugin=mysql_native_password
#启动时候可能报错：[ERROR] RocksDB: Problems validating data dictionary against .frm files, exiting
#[ERROR] RocksDB: Failed to initialize DDL manager.
#此时需要设置rocksdb_validate_tables=2忽略这个错误
rocksdb_validate_tables=2

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
#这里./mysql_install_db需要去执行安装目录下的可执行程序：/usr/local/mysql/scripts/mysql_install_db
#--basedir和--datadir则分别代表mysql的安装目录和数据存放目录
```

- 3、启动数据库

```js
./mysqld_safe --defaults-file=/path/to/my.cnf &

官方推荐在类UNIX系统中使用mysqld_safe脚本来启动mysqld进程，
1. 严重错误产生时自动重启mysqld进程
2. 记录mysqld进程运行信息，保存在错误日志中（error.log，通常在my.cnf中指定）
3. mysqld_safe的启动和运行参数与mysqld通用，对mysqld_safe进程施加参数等同于在mysqld进程上施加参数。
4. mysqld_safe脚本可以启动任何安装方式安装的Mysql,并总是尝试将服务和数据库与工作目录相关联
5. 若每秒启动失败5次，mysqld_safe进程为了防止消耗cpu资源，启动进程将会停顿1s。
```
```js
也可以手动直接启动mysqld进程
/usr/local/mysql/bin/mysqld --defaults-file=/home/zhangyi/mysql-5.6/my.cnf -user=zhangyi
这样mysqld进程就在前台启动，可以cgdb debug
```

- 4、查看数据库启动成功没有

```js
#ps aux |grep mysql
```

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/show_mysql.png)


- 5、登陆mysql

```js
cd /usr/local/mysql/bin
./mysql

可以将mysql执行文件路径加入到PATH环境变量中，这样直接mysql就可以执行了
1、vim ~/.bashrc
2、在文件中增加export PATH=$PATH:/usr/local/mysql/
3、source ~/.bashrc生效

```

![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/start_mysql.tiff)

- 6、也可以直接用MAC上的mycli客户端登陆mysql server

```js
mycli -h mysql_server_ip -u zhangyi
```
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/mac-book-cli.png)

