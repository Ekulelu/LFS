# LFS
使用虚拟机版本进行LFS项目编译，宿主机平台尝试了windows和mac。
Linux为Ubuntu14和16.
在windows下使用VMware搭建虚拟机，这个不要按照默认来安装，否则后面需要添加硬盘。但是在mac上更多问题，由于我使用的是Parallels Desktop 10，按照的是Ubuntu16.导致不能在Ubuntu上安装Visual Tools，这样不能完成从宿主机mac到虚拟机的文件共享。后面改为用Visual box的最新版本安装Ubuntu16，然后再安装Visual tools，再设置共享文件夹，实现粘贴板和文件的共享。

下面开始LFS的尝试。

##安装Ubuntu
第一步是其实在安装Ubuntu的时候对硬盘分好区，lfs建议独立一个分区出来进行项目，以免造成虚拟机的损毁，所以我们在安装虚拟机的时候首先要预留出足够的硬盘空间，我配置的是60G的硬盘，动态增长。然后先分配了4个分区，/ 10g，/home 10g， swap 512M， /lfs 10g(这个分区分出来后不选择挂载目录先)，剩下的空间先预留备用。然后开始安装Ubuntu。
