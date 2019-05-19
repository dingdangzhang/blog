#### myrocks mysql安装后数据库文件存放在哪里的呢？

##### 1、怎么安装myrocks和怎么访问mysql(myrocks)请参考
```js
https://github.com/dingdangzhang/blog/blob/master/how_to_build_myrocks_mysql.md
```
##### 2、通过mycli连接mysql,创建数据库，创建表(table)

```js
create database zhangyi
```
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/creat_database.png)


```js
create table tutorials_tbl_1(
                                         tutorial_id int NOT NULL AUTO_INCREMENT,
                                         tutorial_title VARCHAR(100) NOT NULL,
                                         tutorial_author VARCHAR(40) NOT NULL,
                                         submission_date DATE,
                                         PRIMARY KEY ( tutorial_id )
                                      );
```
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/create_table.png)

##### 3、查询使用的enging

```js
show engines;
```
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/engine.png)

##### 4、直接从mysql中查询一些目录的具体地址

```js
show global variables like "%dir%";
```
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/get_dir.png)

##### 5、在Basedir中有个隐藏目录.rocksdb,里面存放了所有rocksdb的数据文件
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/use_engine.png)

