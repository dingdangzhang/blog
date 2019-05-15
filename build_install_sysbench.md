#### 源码安装sysbench(一个专门用于测试mysql(也可以用去其他测试)工具)
```js
sudo yum -y install mariadb-devel openssl-devel
sudo yum -y install postgresql-devel
git clone git@github.com:akopytov/sysbench.git
cd sysbench
./autogen.sh
# Add --with-pgsql to build with PostgreSQL support
./configure
make -j128
make install
```

#### 查看安装结果
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/sysbench_result.png)
