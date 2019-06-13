#查看NVME盘的sector size
- 可以通过fdisk -l查看
```js
#fdisk -l /dev/nvme2n1

Disk /dev/nvme2n1: 2000.4 GB, 2000398934016 bytes, 3907029168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
- 也可以通过:nvme id-ns /dev/nvme1n1方式
```js
#nvme id-ns /dev/nvme1n1
NVME Identify Namespace 1:
nsze    : 0xe8e088b0
ncap    : 0xe8e088b0
nuse    : 0xe8e088b0
nsfeat  : 0
nlbaf   : 4
flbas   : 0x10
mc      : 0x1
dpc     : 0x1c
dps     : 0
nmic    : 0
rescap  : 0
fpi     : 0
nawun   : 0
nawupf  : 0
nacwu   : 0
nabsn   : 0
nabo    : 0
nabspf  : 0
nvmcap  : 2000398934016
nguid   : 00000000000000000000000000000000
eui64   : 0000000000000000
lbaf  0 : ms:0   ds:9  rp:0 (in use)
lbaf  1 : ms:8   ds:9  rp:0x1
lbaf  2 : ms:64  ds:12 rp:0x1
lbaf  3 : ms:0   ds:12 rp:0
lbaf  4 : ms:8   ds:12 rp:0x1
```
以上inuse表示的ds=9就是512byte ds:12就表示的4K ms:64表示的64byte的元数据，比如可以用于DIF设计

#重新格式化NVME盘
```js
#nvme format /dev/nvme0n1 -l 3  -----这里3表示格式化成4096
Success formatting namespace:1

查看格式化后的sector size
#fdisk -l /dev/nvme3n1

Disk /dev/nvme3n1: 2000.4 GB, 2000398934016 bytes, 488378646 sectors
Units = sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```
