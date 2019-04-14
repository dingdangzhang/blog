##### 运行你自己的程序，比如rocksdb(使用db_bench来跑)

#### 安装火焰图工具，这里直接clone github上的代码即可，不需要编译。直接叫脚本运行即可

```js
git clone git@github.com:brendangregg/FlameGraph.git
cd FlameGraph
```

###### 查看改进程的PID
```js
ps aux |grep process_name
#这里直接使用linux perf工具记录数据
perf record -F 99 -p 30762  -g -- sleep 60  #表示统计一分钟的perf数据，用于后续火焰图画图输入

#制作火焰图
perf script | ./stackcollapse-perf.pl > out.perf-folded
./flamegraph.pl out.perf-folded > perf-kernel.svg
```

###### 将文件perf-kernel.svg拷贝到机器使用浏览器打开即可,比如下面就是一个跑rocksdb的火焰图
![pglog](https://github.com/dingdangzhang/blog/blob/master/file_image/rocksdb_FlameGraph.png)



