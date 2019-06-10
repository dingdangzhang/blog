一般情况下clone rocsdb后按照说明直接make static_lib是没有问题的。但是如果直接make -j32全部build的话(就是要编译起配套的工具如db_bench,test)之类的话，需要依赖gflags gflags-devel glog gtest等。但是我在最新的rocksdb编译时候总是出现

./util/gflags_compat.h:11:26: error: ‘google’ has not been declared

但是rpm -qa |grep gflags趋势安装了的。版本是2.1.2.貌似这个版本比较低了。因此需要从源码来编译安装gflags

1、git clone git@github.com:gflags/gflags.git

2、cd gflags

3、cmake ./CMakeLists.txt

4、make install

然后在编译rocksdb则可以了。make -j128

二、这里可能还存在一个问题就是由于CMake安装目录路径的原因导致编译rocksdb报错。

 

  CCLD     heap_test
util/heap_test.o: In function `__static_initialization_and_destruction_0(int, int)':
/home/boqian.zy/apsara/rocksdb/util/heap_test.cc:20: undefined reference to `google::FlagRegisterer::FlagRegisterer<long>(char const*, char const*, char const*, long*, long*)'
collect2: error: ld returned 1 exit status
make: *** [heap_test] Error 1
make: *** Waiting for unfinished jobs....
 

这里可以把通过rpm安装的cmake卸载，然后在通过源码安装，源码安装也需要i修改默认安装路径：CMAKE_INSTALL_PREFIX=/usr 而不是/usr/local 这样就可以正确编译rocksdb了。

git clone git@github.com:Kitware/CMake.git

cd CMake/

./bootstrap

修改CMAKE_INSTALL_PREFIX=/usr

make clean;make -j128
