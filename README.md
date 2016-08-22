# LFS
使用虚拟机版本进行LFS项目编译，宿主机平台尝试了windows和mac。
Linux为Ubuntu14和16.
在windows下使用VMware搭建虚拟机，这个不要按照默认来安装，否则后面需要添加硬盘。但是在mac上更多问题，由于我使用的是Parallels Desktop 10，按照的是Ubuntu16.导致不能在Ubuntu上安装Visual Tools，这样不能完成从宿主机mac到虚拟机的文件共享。后面改为用Visual box的最新版本安装Ubuntu16，然后再安装Visual tools，再设置共享文件夹，实现粘贴板和文件的共享。

下面开始LFS的尝试。

##1、安装Ubuntu
第一步是其实在安装Ubuntu的时候对硬盘分好区，lfs建议独立一个分区出来进行项目，以免造成虚拟机的损毁，所以我们在安装虚拟机的时候首先要预留出足够的硬盘空间，我配置的是60G的硬盘，动态增长。然后先分配了4个分区，/ 10g，/home 10g， swap 512M， /lfs  格式用了ext4。 10g(这个分区分出来后不选择挂载目录先)，剩下的空间先预留备用。然后开始安装Ubuntu。
安装完Ubuntu之后，你会发现那个没挂载的分区自动挂载了一个目录。打开终端，先用
``` 
passwd root
``` 
命令给root建立一个密码,然后切换到root。先用umount命令将刚刚自动挂载的分区去挂载。然后在/mnt中建立一个lfs目录，然后把这个刚刚的那个分区挂载在这里。再用下面命令打开/etc/fstab.
```
nano /etc/fstab
```
在文件最后面加上，其中sdaX，要安装你的实际情况填写刚刚的那个分区的分区号。
```
/dev/sdaX /mnt/lfs ext4 defaults 0 2  
```
用ctrl+x退出nano，选择yes后保存更改。
这样就能在系统进入后自动挂载这个分区到/mnt/lfs

宿主机linux的系统需求，lfs需要宿主机安装了一些软件，但是Ubuntu的发行版本并不是所有都有。好在这些软件都可以在lfs的packages里面找到。所以建议先下载lfs的packages，下载点在官网的lfs的download页面下面的几个链接中。这里使用7.9systemd版本。
Ubuntu缺的几个东西，请按照下面的顺序安装，因为有依赖关系。

M4，

bison，

yacc是bison的硬链接，

gawk，

安装完后需要调整3个链接
/bin/sh -> /usr/bin/bash # 这里 sh 是到 bash 的硬链接
yacc is bison (GNU Bison) 3.0.4 # yacc 是到 bison 的硬链接
/usr/bin/awk -> /usr/bin/gawk # awk 是到 gawk 的硬链接


##2、环境配置
继续用管理员root账号输入下面代码建立一个环境变量
```
export LFS=/mnt/lfs
```
建立文件夹tools和sources
```
mkdir $LFS/tools  $LFS/sources
```
创建/tools软链接，将其指向 LFS 分区中新建的文件夹。创建的符号链接使得编译的工具链总是指向 /tools 文件夹,也就是说编译器、汇编器以及链接器在第五章中（我们仍然使用宿主机的一些工具的时候）和下一章中（当我们 “chrooted” 到 LFS 分区时）都可以工作。
```
ln -sv $LFS/tools /
```

建立lfs用户和lfs用户组，用来隔离开linux用户
```
groupadd lfs
useradd -s /bin/bash -g lfs -m -k /dev/null lfs
```
给lfs用户设立密码
```
passwd lfs
```
更改tools和sources的用户，确保lfs用户可以在sources和tools下面做事情
```
chown -v lfs $LFS/tools
chown -v lfs $LFS/sources
```
用
```
su - lfs
```
登陆lfs用户，配置他的bash shell环境，分别针对登录shell和非登录shell的情况。创建一个干净的构建环境
```
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```
```
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/tools/bin:/bin:/usr/bin
export LFS LC_ALL LFS_TGT PATH
EOF
```
最后启用更改的配置文件
```
source ~/.bash_profile
```

编译的时候可以用make -j2来指定编译时候所用的处理器个数。2代表2个。

##3、构建临时系统
这部分对应的是教程的第五章
开始前最后确认几样东西，虽然说可能没设置也能编译过去，但是万一出问题了，那就很难找到源头所在，所以还是安装要求乖乖来。
$LFS=/mnt/lfs
shell 使用的是 bash。
sh 是到 bash 的符号链接。
/usr/bin/awk 是到 gawk 的符号链接。
/usr/bin/yacc 是到 bison 的符号链接或者一个执行 bison 的小脚本。

将下载到的packages压缩包解压到/mnt/lfs/sources文件夹下。
从下面开始，安装每一个软件都是按照下面的步骤先进行
1、确认是在sources目录下
2、用tar命令解压要安装的软件包
3、cd到刚刚解压出来的软件包的目录内
4、根据安装这个软件的教程，安装这个软件。一般是会在软件包目录下再建立一个build文件夹
5、回到到sources目录。除非特别说明，用rm删除掉刚刚解压的软件包文件夹。

###Binutils-2.26 - Pass 1
安装说明书来，没问题通过。

###GCC-5.3.0 - Pass 1
这里犯了个大错，忽略了最开始的几行命令，以为在宿主机安装了mpfr、gmp、mpc就行了
需要老老实实在gcc-5.3.0目录下执行下面命令
```
tar -xf ../mpfr-3.1.3.tar.xz
mv -v mpfr-3.1.3 mpfr
tar -xf ../gmp-6.1.0.tar.xz
mv -v gmp-6.1.0 gmp
tar -xf ../mpc-1.0.3.tar.gz
mv -v mpc-1.0.3 mpc
```
然后再继续按照说明来，终于可以编译成功了。

###Linux-4.4.2 API Headers
两条命令，别忘记进入linux-4.4.2目录

###Glibc-2.23
配置文件说找不到gawk，但是按照前面的检测是可以检测/usr/bin/awk到指向/usr/local/bin/gawk-4.1.3的。以为是gawk没安装好，又安装了一遍，结果还是不行。不得已，翻glibc的configure文件，发现里面用到$AWK来判断是否有gwak！然后一查echo $AWK,空值。。。所以找不到/usr/bin/awk，进而更找不到/usr/local/bin/gawk-4.1.3
自己加上，注意AWK要大写，否则没用。
```
export AWK=/usr/bin/awk
```
编译通过，安装通过。

然后进行测试，通过！！

###Libstdc++-5.3.0
这个软件包是在gcc里面的，所以要解压的是gcc-5.3.0。然后进入该目录下执行命令。


###Binutils-2.26 - Pass 2
通过

###GCC-5.3.0 - Pass 2
第一次编译失败，报了两个错，这两个错像是没把三个依赖软件放置好的错误。再回去上一步的测试，输出还是没问题的。从检测开始重新过一遍，到gcc，这次不用多线程编译。第二次成功了。测试通过！！

###Tcl-core-8.6.4
通过

###Expect-5.45
通过

###DejaGNU-1.5.3
通过

###Check-0.10.0
check耗时较长，可以不测了。通过

###Ncurses-6.0
通过

###Bash-4.3.30
通过

###Bzip2-1.0.6
通过

###Coreutils-8.25
不要测试，耗时超级长，1个小时还没结果，然后卡死，重启了

###Diffutils-3.3
下面的都不测试了，通过

###File-5.25
通过

###Findutils-4.6.0
通过

###Gawk-4.1.3
通过

###Gettext-0.19.7
通过

###Grep-2.23
通过

###Gzip-1.6
通过

###M4-1.4.17
通过

###Make-4.1
通过

###Patch-2.7.5
通过

###Perl-5.22.1
通过

###Sed-4.2.2
通过

###Tar-1.28
通过

###Texinfo-6.1
通过

###Util-linux-2.27.1
通过

###Xz-5.2.2
通过

因为硬盘还有5.8G，所以不清理了

改变tools目录所有者
```
chown -R root:root $LFS/tools
```

到此，第五章工作结束，下面开始构建LFS系统。


