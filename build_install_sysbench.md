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
sysbench /usr/local/share/sysbench/oltp_read_write.lua   --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=zhangyi --mysql-password=zhangyi --mysql-db=zhangyi --tables=10 --table-size=1000 --report-interval=1 --threads=1 --time=30 prepare
#这里lua脚本默认是使用的innodb的存储引擎，这里需要修改/usr/local/share/sysbench/oltp_common.lua脚本中innodb->rocksd
```
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/lua.png)

这里仅做数据库测试，其他测试可以是用sysbench –help，sysbench cpu help等查看相应参数。
OLTP的测试流程分为3个阶段：1、建测试表及数据；2、进行测试；3清除数据。（1、prepare；2、run；3、cleanup）

```js
sysbench /usr/local/share/sysbench/oltp_read_write.lua   --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=zhangyi --mysql-password=zhangyi --mysql-db=zhangyi --tables=10 --table-size=1000 --report-interval=1 --threads=1 --time=30 prepare
sysbench /usr/local/share/sysbench/oltp_read_write.lua   --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=zhangyi --mysql-password=zhangyi --mysql-db=zhangyi --tables=10 --table-size=1000 --report-interval=1 --threads=1 --time=30 run
sysbench /usr/local/share/sysbench/oltp_read_write.lua   --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=zhangyi --mysql-password=zhangyi --mysql-db=zhangyi --tables=10 --table-size=1000 --report-interval=1 --threads=1 --time=30 cleanup
```
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/prepare.png)
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/run.png)
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/cleanup.png)
