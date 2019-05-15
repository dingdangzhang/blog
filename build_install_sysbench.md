#### 1、源码安装sysbench(一个专门用于测试mysql(也可以用去其他测试)工具)
```js
sudo yum -y install mariadb-devel openssl-devel
sudo yum -y install postgresql-devel
git clone git@github.com:akopytov/sysbench.git
cd sysbench
git checkout -b 1.0 origin/1.0
./autogen.sh
# Add --with-pgsql to build with PostgreSQL support
./configure
make -j128
make install
```

#### 2、查看安装结果
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/sysbench_result.png)

#### 3、如果安装的mysql不是标准的目录
```js
./configure \
--with-mysql-includes=/opt/mysql/include/mysql #  表示mysql头文件目录  \
--with-mysql-libs=/opt/mysql/lib/mysql #  表示mysql模块目录
```

编译成功之后, 就要开始测试各种性能了, 测试的方法官网网站上也提到一些, 但涉及到 OLTP 测试的部分却不够准确. 在这里我大致提一下

1	cpu性能测试
```js
sysbench --test=cpu --cpu-max-prime=20000 run cpu测试主要是进行素数的加法运算，在上面的例子中，指定了最大的素数为 20000，自己可以根据机器cpu的性能来适当调整数值。
```
2	线程测试
```js
sysbench --test=threads --num-threads=64 --thread-yields=100 --thread-locks=2 run
```
3	磁盘IO性能测试
```js
sysbench --test=fileio --num-threads=16 --file-total-size=3G --file-test-mode=rndrw prepare
sysbench --test=fileio --num-threads=16 --file-total-size=3G --file-test-mode=rndrw run
sysbench --test=fileio --num-threads=16 --file-total-size=3G --file-test-mode=rndrw cleanup
```
上述参数指定了最大创建16个线程，创建的文件总大小为3G，文件读写模式为随机读

4	内存测试
```js
sysbench --test=memory --memory-block-size=8k --memory-total-size=4G run
```
上述参数指定了本次测试整个过程是在内存中传输 4G 的数据量，每个 block 大小为 8K。

5	OLTP测试
```js
sysbench --test=oltp --mysql-table-engine=myisam --oltp-table-size=1000000 \
--mysql-socket=/tmp/mysql.sock --mysql-user=test --mysql-host=localhost \
--mysql-password=test prepare
```
上述参数指定了本次测试的表存储引擎类型为 myisam，这里需要注意的是，官方网站上的参数有一处有误，即 --mysql-table-engine，官方网站上写的是 --mysql-table-type，这个应该是没有及时更新导致的。另外，指定了表最大记录数为 1000000，其他参数就很好理解了，主要是指定登录方式。测试 OLTP 时，可以自己先创建数据库 sbtest，或者自己用参数 --mysql-db 来指定其他数据库。--mysql-table-engine 还可以指定为 innodb 等 MySQL 支持的表存储引擎类型。
