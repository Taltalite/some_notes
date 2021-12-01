# 如何将 UNCALLED 迁移到 Xavier(arm) 平台

## 1. Compiling BWA for the Arm64 architecture

UNCALLED 目录下有 submods 文件夹，里面包含了其使用的 bwa 项目文件

```bash
cd submods/bwa/
sed -i -e 's/<emmintrin.h>/"sse2neon.h"/' ksw.c
wget https://gitlab.com/arm-hpc/packages/uploads/ca862a40906a0012de90ef7b3a98e49d/sse2neon.h
make clean all
```

重新 make bwa 项目

## 2. 更改 setup.py 文件

找到 UNCALLED 目录下的 setup.py 文件稍作更改，将

```python
subprocess.check_call([
                "./configure", 
                    "--enable-threadsafe", 
                    "--disable-hl",
                    "--prefix", hdf5_dir,
                    "--enable-shared=no",
                    "--with-pic=yes"
            ])
```

改为：

```python
subprocess.check_call([
                "./configure", 
                    "--enable-threadsafe", 
                    "--disable-hl",
                    "--prefix", hdf5_dir,
                    "--enable-shared=no",
                    "--with-pic=yes",
                    "--build=arm-linux"
            ])
```

## 3.更改 Makefile 文件生成动态库

进入 submods/bwa/ 目录，找到 Makefile 文件，做如下更改：

```makefile
CFLAGS= 	-g -Wall -Wno-unused-function -O2 -fPIC

bwa:libbwa.so $(AOBJS) main.o
		$(CC) $(CFLAGS) $(DFLAGS) $(AOBJS) main.o -o $@ -L. -lbwa $(LIBS)

bwamem-lite:libbwa.so example.o
		$(CC) $(CFLAGS) $(DFLAGS) example.o -o $@ -L. -lbwa $(LIBS)

# libbwa.a:$(LOBJS)
# 		$(AR) -fPIC -csru $@ $(LOBJS)

libbwa.so:$(LOBJS)
		$(CC) -shared -fPIC $(LOBJS) -o $@
```

将 libbwa.a 部分注释或者删除

## 4.复制 Makefile 文件

将上述更改的 Makefile 文件复制一份到 src/ 目录下，替换原来的 Makefile_bwa

## 5.运行 setup.py 脚本

## 6.设置动态库链接

如果上述步骤没有报错，且 setup.py 脚本提示 uncalled 安装成功，先使用 uncalled 命令验证是否真正完成了

```
(base) bio-lab@biolab-desktop:~$ uncalled
usage: uncalled [-h] {index,map,realtime,sim,pafstats} ...

Rapidly maps raw nanopore signal to DNA references

positional arguments:
  {index,map,realtime,sim,pafstats}
    index               Builds the UNCALLED index of a FASTA reference
    map                 Map fast5 files to a DNA reference
    realtime            Perform real-time targeted (ReadUntil) sequencing
    sim                 Simulate real-time target sequencing.
    pafstats            Computes speed and accuracy of UNCALLED mappings.

optional arguments:
  -h, --help            show this help message and exit
```

出现如上信息则表示成功，如果报错找不到 libbwa.so 动态库则根据以下步骤：

首先找到 uncalled 安装路径，以本文为例是 /home/bio-lab/.local/lib/python3.8/site-packages/uncalled-2.2-py3.8-linux-aarch64.egg

然后使用命令 ldd :

```bash
ldd _uncalled.cpython-38-aarch64-linux-gnu.so
```

查看动态库链接情况，出现下面的输出：

```
linux-vdso.so.1 (0x0000007f7b2d1000)
libbwa.so => not found
libz.so.1 => /home/bio-lab/nvme/anaconda3/lib/libz.so.1 (0x0000007f7adf8000)
libdl.so.2 => /lib/aarch64-linux-gnu/libdl.so.2 (0x0000007f7ade3000)
libstdc++.so.6 => /home/bio-lab/nvme/anaconda3/lib/libstdc++.so.6 (0x0000007f7ac54000)
libm.so.6 => /lib/aarch64-linux-gnu/libm.so.6 (0x0000007f7ab9b000)
libgcc_s.so.1 => /home/bio-lab/nvme/anaconda3/lib/libgcc_s.so.1 (0x0000007f7ab78000)
libpthread.so.0 => /lib/aarch64-linux-gnu/libpthread.so.0 (0x0000007f7ab4c000)
libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000007f7a9f3000)
/lib/ld-linux-aarch64.so.1 (0x0000007f7b2a5000)
```

那么将上面步骤中 make 生成的 libbwa.so 文件路径写入 /etc/ld.so.conf

```bash
sudo vim /etc/ld.so.conf
```

以本文为例，在文末添加一行：

```
/home/bio-lab/nvme/UNCALLED/submods/bwa
```

再次使用 ldd 命令

```bash
ldd _uncalled.cpython-38-aarch64-linux-gnu.so
```

出现下面的输出则说明 uncalled 可以正常运行：

```
linux-vdso.so.1 (0x0000007f7b2d1000)
libbwa.so => /home/bio-lab/nvme/UNCALLED/submods/bwa/libbwa.so (0x0000007f7ae23000)
libz.so.1 => /home/bio-lab/nvme/anaconda3/lib/libz.so.1 (0x0000007f7adf8000)
libdl.so.2 => /lib/aarch64-linux-gnu/libdl.so.2 (0x0000007f7ade3000)
libstdc++.so.6 => /home/bio-lab/nvme/anaconda3/lib/libstdc++.so.6 (0x0000007f7ac54000)
libm.so.6 => /lib/aarch64-linux-gnu/libm.so.6 (0x0000007f7ab9b000)
libgcc_s.so.1 => /home/bio-lab/nvme/anaconda3/lib/libgcc_s.so.1 (0x0000007f7ab78000)
libpthread.so.0 => /lib/aarch64-linux-gnu/libpthread.so.0 (0x0000007f7ab4c000)
libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000007f7a9f3000)
/lib/ld-linux-aarch64.so.1 (0x0000007f7b2a5000)
```
