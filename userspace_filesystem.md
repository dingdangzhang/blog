用户态文件系统

1、Mkfs就是在物理硬盘上将superblock写好，并且能通过这些信息可以恢复出一个所谓的fs实例。格式化完成后，在启动(mount)这个文件系统则可以通过replay方式将这个文件系统回放出来。

2、mount操作就是读取硬盘上的superblock信息 然后恢复一个内存的文件系统fs信息在内存中。

3、create dir操作就是在fs内存结构(dir_map)上增加记录dir目录信息。同时操作需要通过transanction记录之记录日志文件

4、create file操作就是在fs内存数据结构中的dir_map结构下下的dir->file_map下创建文件信息。二这个文件信息就是用fnode或者inode来记录的。这个结构就是文件的关键结构。记录文件大小，创建时间，包含的空间extent（offset和length）信息。并且在创建这个文件的时候也会进行transanction 日志的持久化操作，从而保证数据可以恢复和回滚。然后这个file还对应了这个文件对应的真正的设备bdev指针用于后续读写改文件。

5、如果对文件进行写操作

5.1、覆盖写(不支持)

- 5.2、append写,则每次写文件时候，其实是对这个文件进行重新冲头开始写。一般场景应为为新建一个文件,因为这个文件已经在创建的时候就分配了大小。所以这里一般需要在创建一个filewriter对象来记录这个文件的写的数据，这个数据暂时append到这个filewrite的内存buffer中。然后等待flush和fsync操作。

- - 5.2.1、filewriter的flush操作：就是将这个inode对应的file的 filewriter->buffer的内容通过这个file对应的bdev的write接口写到盘上去。

6、记住这里有两个概念在Linux需要区分：

Flush是指的将文件的内容刷到文件系统的page cache中

Fsync则是将page cache的数据刷到盘上。

只是在用户态文件系统中由于没有page cache的概念的所以flush就指的刷到dev上。而fsync则是将dev上的数据刷到盘上。但是这里dev不暂存数据因此其实flush的时候就刷到了盘上。而fsync没有实际操作。

---------------------

Mkfs:
1、初始化分配器，也就是把整个盘的划分成1M1M的块，然后初始化到free数组中。方便后续分配
2、根据盘的大小初始化块supperblock信息。superblock信息包含：盘名称，版本，blocksize,产生一个uuid,
3、写supplelock信息到盘上
4、上述步骤中途有初始化free块信息，这些需要log写入到日志文件中，所以这里还有一步写日志操作，而写日志操作封装了很多命令的transanction操作。

mount操作
1、读取指定盘上的supperblock信息。解码supperblock信息,通过解码出来的supper block信息填充改文件系统对应的内存结构信息。
2、初始化一个空的allocated空间分配器，方便后续分配使用
3、replay操作：
   1、读取supper信息，解析出uuid等信息
   2、然后解析log信息的每条record信息 得到之前持久化的每条信息。然后根据每条信息去解析每条transanction record的详细内容。
   3、通过解析来还原当前文件系统结构题中的类似alloc 块的信息，比如那些分配(alloc[id]->init_add_free)了，那些是删除了(alloc[id]->init_rm_free)的的.以及哪些文件(file_map)是进行了flink操作的，那些目录是create的(dir_map)。通过这些解析信息更新内存结构体信息
4、至此整个盘对应的文件系统就全部恢复到了内存中。

Create volume:
1、如果整个文件系统是第一次创建volume的话，需要为改文件系统创建一个volume_dir和volume_supper_dir目录，整个文件系统只会有这个两个目录，其中之后创建的volume都会放在volume_dir目录下。
2、就是在几个之前创建的母盘文件系统基础上去找一个文件系统来创建一个目录，这些dir放在fs文件系统内存结构体的map[]数组中。
3、虽然这里创建了一个dir,但是这个只是修改了一个内存结构体信息，所以如果要持久化就需要使用log的方式讲改次修改内存信息持久化到盘上，也就是使用一条transanction语意讲改次创建动作记录到log.写盘。
4、真正创建volume的动作实在create file。这里一个volume就是一个file。然后首先是在这个volume_dir目录下(dir->file_map)查找时候发存在这个volume名字的文件是否存在。如果不存在创建一个file,这个file包含一个结构体叫fnode，这个fnode记录文件创建的时间，大小，以及对应的bdev设备等等信息.然后将这个file放在他的dir结构体成员变量file_map中保存。
5、同样因为这里创建了一个新的文件，所以这里需要将改次创建文件的操作通过日志持久化(log_t.op_raw_file_update(raw_file->fnode)).持久化信息是fnode信息,同时将改fnode信息和这个dir目录和volume名字联系在一起表示这个fnode表示的就是这个目录下的volume文件，这个也是通过一个log日志操作持久化的(log_t.op_file_link(dirname, filename, file->fnode->ino))。
6、最后一步最重要的，就是将这个文件格式化成一个新的文件系统，然后对这个文件再走set_super_set，add_block_extent操作+mkfs操作。

rocksdb初始化：
1、Rocksdb初始化时候将制定的盘以及这个盘上的volume信息告诉rokcsdb,然后就会将这个盘上的volume对应的文件系统加载到rocksdb实例空间上去，让rocksdb可以直接访问这个volume对应的文件系统。

写操作：
1、首先读写操作前需要先在rocksdb中创建一个writer或者reader,读写操作是先看这个dir+file在用户台文件系统中是否存在，如果不存在就在文件系统中创建一个file然后添加到文件系统dir->file_map中，如果文件存在则更新文件fnode信息，如更改时间等
2、写的话是先写到filewirter的append buffer上去，然后如果buffersize > 512K 后自动flush buffer中的数据到盘上，否则内存中有512K的数据可能丢失。
3、读的话就是直接从盘上去读，首先通过offset去对应的文件上seek到对应的偏移，然后直接从盘read数据返回。


