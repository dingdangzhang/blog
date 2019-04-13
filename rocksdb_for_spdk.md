#### rocksdb继承spdk用户态文件系统

##### 下载最新的SPDK

```js
git clone https://github.com/spdk/spdk.git
cd spdk
./configure
make
```

##### 下载SPDK对应维护的继承了SPDK env的rocksdb代码，注意不是fb维护的。


```js
cd ..
git clone -b spdk-v5.6.1 https://github.com/spdk/rocksdb.git
```
##### 编译rcoskdb

- 注意：这里如果直接make -j128可能会因为rocksdb make all时候会编译test，导致编译报错，这里可以在makefile中先屏蔽到test

```js
cd rocksdb
make db_bench SPDK_DIR=path/to/spdk
```
##### 设置SPDK大页(至少设置5G)

```js
spdk目录下的脚本
HUGEMEM=5120 scripts/setup.sh
```

##### SPDK设置盘符

```js
scripts/gen_nvme.sh > /usr/local/etc/spdk/rocksdb.conf
```

##### 为需要跑的盘格式化文件系统(比如:Nvme1n1)

```js
spdk目录下的工具
test/blobfs/mkfs/mkfs /usr/local/etc/spdk/rocksdb.conf Nvme0n1
```

##### 创建一个rocksdb flagfile参数文件

```js
cd rocksdb
touch db_bench_flagfile
input:

--disable_seek_compaction=1
--mmap_read=0
--statistics=1
--histogram=1
--key_size=16
--value_size=1000
--block_size=4096
--cache_size=0
--bloom_bits=10
--cache_numshardbits=4
--open_files=500000
--verify_checksum=1
#--db=/mnt/rocksdb
--sync=0
--compression_type=none
--stats_interval=1000000
--compression_ratio=1
--disable_data_sync=0
--target_file_size_base=67108864
--max_write_buffer_number=3
--max_bytes_for_level_multiplier=10
--max_background_compactions=10
--num_levels=10
--delete_obsolete_files_period_micros=3000000
--max_grandparent_overlap_factor=10
--stats_per_interval=1
--max_bytes_for_level_base=10485760
--benchmarks=fillseq
--threads=1
--disable_wal=1
--use_existing_db=0
--num=100000000
--spdk=/usr/local/etc/spdk/rocksdb.conf
--spdk_bdev=Nvme1n1
--spdk_cache_size=4096
--statistics=1
--histogram=1
--stats_interval=1
--stats_interval_seconds=1
--stats_per_interval=0
--stats_dump_period_sec=60

```

##### 运行rocksdb

```js
./db_bench --flagfile=/home/zhangyi/zhangyi_flag.txt
```

