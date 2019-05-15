#### 自动挂载分区到指定目录下

系统重启后自动挂载mount目录
##### 比如如下两个挂载点
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/old_mount_dir.png)

##### 修改/etc/fstab文件 增加如下内容
![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/modify-fstab.png)

##### reboot机器

##### 查看自动mount点
 ![ceph架构](https://github.com/dingdangzhang/blog/blob/master/file_image/auto-mount.png)

