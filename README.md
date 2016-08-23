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
Ubuntu缺的几个东西，请按照下面的顺序安装，因为有依赖关系。这些软件可以在7.9systemd的包里面找到，也可以在线安装。

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
这部分对应的是教程的第五章。编译的时候如果有make命令，请加上make -j8，使用多核编译。
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


##第六章 建立LFS系统
如果之前有建立make flag，进入到这个环境会没有用了。可以用make -j8指令。其他的check也要加上这个-j8，除了特殊说明的外
准备虚拟内核文件系统
```
mkdir -pv $LFS/{dev,proc,sys,run}
```
建立原始设备节点
```
mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3
```
挂载设备
```
mount -v --bind /dev $LFS/dev
```
挂载文件系统
```
mount -vt devpts devpts $LFS/dev/pts -o gid=5,mode=620
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run
```
在某些宿主机系统里，/dev/shm 是一个指向 /run/shm 的软链接。这个 /run 下的 tmpfs 文件系统已经在之前挂载了，所以在这里只需要创建一个目录。
```
if [ -h $LFS/dev/shm ]; then
  mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
```
进入chroot环境
```
chroot "$LFS" /tools/bin/env -i \
HOME=/root \
TERM="$TERM" \
PS1='\u:\w\$ ' \
PATH=/bin:/usr/bin:/sbin:/usr/sbin:/tools/bin \
/tools/bin/bash --login +h
```
这里会把/mnt/lfs作为根目录，也就没有了mnt/lfs/sources，/sources就是原来那个mnt/lfs/sources目录。

建立文件目录
```
mkdir -pv /{bin,boot,etc/{opt,sysconfig},home,lib/firmware,mnt,opt}
mkdir -pv /{media/{floppy,cdrom},sbin,srv,var}
install -dv -m 0750 /root
install -dv -m 1777 /tmp /var/tmp
mkdir -pv /usr/{,local/}{bin,include,lib,sbin,src}
mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man}
mkdir -v /usr/{,local/}share/{misc,terminfo,zoneinfo}
mkdir -v /usr/libexec
mkdir -pv /usr/{,local/}share/man/man{1..8}
case $(uname -m) in
x86_64) ln -sv lib /lib64
ln -sv lib /usr/lib64
ln -sv lib /usr/local/lib64 ;;
esac
mkdir -v /var/{log,mail,spool}
ln -sv /run /var/run
ln -sv /run/lock /var/lock
mkdir -pv /var/{opt,cache,lib/{color,misc,locate},local}
```
建立必要的文件和软连接
```
ln -sv /tools/bin/{bash,cat,echo,pwd,stty} /bin
ln -sv /tools/bin/perl /usr/bin
ln -sv /tools/lib/libgcc_s.so{,.1} /usr/lib
ln -sv /tools/lib/libstdc++.so{,.6} /usr/lib
sed 's/tools/usr/' /tools/lib/libstdc++.la > /usr/lib/libstdc++.la
ln -sv bash /bin/sh
```

```
ln -sv /proc/self/mounts /etc/mtab
```
建立/etc/passwd和/etc/group，代码不贴了
执行
改变目录权限
开始安装软件包

###Linux-4.4.2 API Headers
通过
###Man-pages-4.04
通过
###Glibc-2.23
有几个错误是会报错的

###调整工具链

###Zlib-1.2.8
通过

###File-5.25
通过

###Binutils-2.26
check时候出错
```
Makefile:3546: recipe for target 'check-DEJAGNU' failed
make[5]: *** [check-DEJAGNU] Error 1
make[5]: Leaving directory '/sources/binutils-2.26/build/ld'
Makefile:1866: recipe for target 'check-am' failed
make[4]: *** [check-am] Error 2
make[4]: Leaving directory '/sources/binutils-2.26/build/ld'
Makefile:1713: recipe for target 'check-recursive' failed
make[3]: *** [check-recursive] Error 1
make[3]: Leaving directory '/sources/binutils-2.26/build/ld'
Makefile:1868: recipe for target 'check' failed
make[2]: *** [check] Error 2
make[2]: Leaving directory '/sources/binutils-2.26/build/ld'
Makefile:7211: recipe for target 'check-ld' failed
make[1]: *** [check-ld] Error 2
make[1]: Leaving directory '/sources/binutils-2.26/build'
Makefile:2198: recipe for target 'do-check' failed
make: *** [do-check] Error 2
```
不知道继续进行下去会不会有问题。继续下去。回去查了看官方的log，也是有这个错误。

###GMP-6.1.0
通过

###MPFR-3.1.3
通过

###MPC-1.0.3
通过

###GCC-5.3.0
这个也会报几个错，但是说明书上说是会报错的。具体的错误请参照
[http://www.linuxfromscratch.org/lfs/build-logs/7.9/i7-5820K/test-logs/082-gcc-5.3.0]

###Bzip2-1.0.6
通过

###Pkg-config-0.29
通过

###Ncurses-6.0
通过

###Attr-2.4.47
通过，这里有一处不能用多核编译的。

###Acl-2.2.52
通过

###Libcap-2.25
通过

###Sed-4.2.2
通过

###Shadow-4.2.1
通过，这个安装后要设定密码

###Psmisc-22.21
通过

###Procps-ng-3.3.11
通过

###E2fsprogs-1.42.13
通过

###Iana-Etc-2.30
通过

###M4-1.4.17
会报一个错误，test-update-copyright.sh，说明书上说可以忽略

###Bison-3.0.4
通过

###Flex-2.6.0
通过

###Grep-2.23
通过

###Readline-6.3
通过

###Bash-4.3.30
通过

###Bc-1.06.95
通过

###Libtool-2.4.6
check会有几个错误。但可以忽略

###GDBM-1.11
通过

###Expat-2.1.0
通过

###Inetutils-1.9.4
多核编译下貌似会出错。第二次没用多核编译通过了。

###Perl-5.22.1
通过

###XML::Parser-2.44
通过

###Autoconf-2.69
会报6个错，4个expected fails。其他两个错不管。

###Automake-1.15
通过，check特别慢

###Coreutils-8.25
通过

###Diffutils-3.3
test-update-copyright.sh这个会报错，不用管

###Gawk-4.1.3
通过

###Findutils-4.6.0
通过

###Gettext-0.19.7
检查9个报错，因为缺少依赖。不用理。

###Intltool-0.51.0
通过

###Gperf-3.0.4
通过

###Groff-1.22.3
说明书中的<paper_size>要改为A4
```
PAGE=A4 ./configure --prefix=/usr
```
多核编译出错，使用单核编译可以通过。


###Xz-5.2.2
通过

###GRUB-2.02~beta2
通过

###Less-481
通过

###Gzip-1.6
通过

###IPRoute2-4.4.0
通过

###Kbd-2.0.3
通过

###Kmod-22
通过

###Libpipeline-1.4.1
通过

###Make-4.1
通过

###Patch-2.7.5
通过

###Sysklogd-1.5.1
7.9的tar包中没有这个，需要自己用单独的链接下，文章开头有链接。通过。

###Sysvinit-2.88dsf
7.9的包里面也没这个，需要下载它和它的补丁。补丁那个链接用IE打开才能下载，chrome会直接变为打开。
通过

###Tar-1.28
通过

###Texinfo-6.1
通过

###Eudev-3.1.5
也是没有，另外下。这里还会缺少另外一个包udev-lfs-20140408.tar.bz2。
通过

###Util-linux-2.27.1
通过

###Man-DB-2.7.5
通过

###Vim-7.4
通过

###


###

###


###

###

###

###

###

###

###

###
