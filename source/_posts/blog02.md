---
title: 随笔：在超算-计算集群中编译安装OpenFOAM-5.x
date: 2022-11-30 16:07:53
tags: [偏微分方程数值解, CFD, 随笔]
index_img: /img/tianmen.jpeg
banner_img: /img/tianmen.jpeg
comment: 'valine'
---

# 安装的准备
现在OpenFOAM已经更新到了第十版，之所以安装5版本，是因为我个人现在做CFD-DEM耦合的工作，CFDEM耦合软件支持的OpenFOAM就是5.x版本。不管是哪一个版本，安装过程和准备文件是基本一致的。
安装过程主要参考如下博客和帖子：
[https://www.cfd-china.com/topic/4296/](https://www.cfd-china.com/topic/4296/)一种集群非root用户编译openfoam的方法-基于centos7
[https://blog.csdn.net/weixin_41734903/article/details/105125214](https://blog.csdn.net/weixin_41734903/article/details/105125214)
[https://blog.csdn.net/weixin_42230462/article/details/115555172](https://blog.csdn.net/weixin_42230462/article/details/115555172)

## 文件清单
采用编译安装方式，首要就是准备好OpenFOAM的源代码，这个通过git clone或者下载官网上的源码压缩包就可以了。
```bash
git clone git://github.com/OpenFOAM/OpenFOAM-5.x.git
git clone git://github.com/OpenFOAM/ThirdParty-5.x.git
```
由于在计算集群（超算）中，个人用户是没有安装底层库的权限的，且集群不连接外网，需要用户自己将一些必要的库准备好，然后传上去。超算平台和大型计算平台主要采用CentOS7系统，接下来的操作都以该系统为例。
如果自己不清楚自己的系统版本，可以用uname -a命令查看系统版本，lsb_release -a可以查看属于什么操作系统。
首先需要在自己的个人电脑上面安装一个与集群上系统一致的本地Linux虚拟机，个人可以使用VMware workstation play免费创建虚拟机。虚拟机的安装过程比较简单，不再赘述。
在本地虚拟上，首先换源，换成国内源，下载软件速度快。然后安装yum-utils包，用来下载必须的依赖库。
```bash
su root #在本地虚拟机上输入密码，切换成为root用户
yum -y install yum-utils
```
然后下载OpenFOAM的依赖库，比较关键的有：binutils、boost、bison、flex、glibc、hwloc、m4、libtool、zlib，cmake这几个库的安装方式一致，以binutils为例：
```bash
cd ~/
mkdir packages #建立一个文件夹统一存放这些库
cd packages && mkdir binutils #在packages下面建立一个文件夹存放binutils库
yumdownloader binutils #下载这个库的软件包
#如果所有的库都按照这个顺序下载好了，可以先打包上传到集群中
#接下来就是解压库，然后将路径加入到bashrc，这里还是先在本地虚拟机操作，成功安装这些库，然后再在超算上实现一次
cd binutils
rpm2cpio binutils-2.27-44.base.el7_9.1.x86_64.rpm | cpio -idvm #解压包，然后可以在binutils文件夹下面发现多出一个usr文件夹
vim ~/.bashrc
export PATH=$PATH:$HOME/packages/binutils/usr/bin #将库文件目录加入到系统目录中，至此，这个库已经装好了

#重复以上步骤，依次安装所有的库，现在本地安装通过，再在超算上重复操作。保持两者一致性。
```
这里有一个值得一说的点，由于CentOS7默认的gcc版本是4.85，刚好符号OpenFOAM-5.x安装要求，我并没有再安装gcc，gcc版本太旧或者太新都会导致编译出错。
然后是安装openmpi，从安装角度，这个是OpenFOAM最推荐使用的并行库，且集群中，最好只有这个并行库，mpich可能会和openmpi起冲突。如果使用intelmpi，会在AMD核心集群上存在兼容性问题。如果集群中安装了别的并行库，最好先卸载掉。检查OpenFOAM-5.x使用的是openmpi2.1版本，从Openmpi官网上下载源码，在本地和集群中都保留一份。
```bash
cd ~/
mkdir openmpi #建立openmpi的安装路径
cd OpenFOAM
cd Third-party-5.x/ #最好将openmpi源码解压到此文件夹下，因为可能会有一些找不到文件的错误
tar -zxvf openmpi-2.1.1.tar.gz #解压openmpi的源码
cd openmpi-2.1.1/
./configure --prefix=/home/username/openmpi/ #配置openmpi
make && make install #耐心等待安装完成
```
username是自己的账户名，不是就打一个"username"就可以了
如果在openmpi编译过程中，出现了“****  -l和-lr之间没有空格”这个编译错误，是openmpi的一个bug，将配置语句改成
```bash
./configure --prefix=/home/username/openmpi --with-ucx=/usr
```
就可以正常编译，然后没有错误以后，将openmpi的路径加入到系统路径中：
```bash
vim ~/.bashrc
export PATH=$PATH:$HOME/openmpi/bin
export LD_LIBRARY_PATH=/home/username/openmpi/lib:$LD_LIBRARY_PATH
source ~/.bashrc #让配置文件生效
```
如果没有报错，可以运行一下mpicc或者which mpicc，如果没有出现mpicc not found这样的错误，并且显示了mpicc的路径就是安装成功了。

# 编译OpenFOAM-5.x
到这里，我个人建议，是本地虚拟机和计算集群都执行一样的操作，保持一致性。这样也方便以后自己程序的调试，至于还有一个作用，在文末可以看到。
首先，需要将OpenFOAM的环境说明写入系统.bashrc文件里面：
```bash
source $HOME/OpenFOAM/OpenFOAM-5.x/etc/bashrc WM_LABEL_SIZE=64 WM_COMPILER_TYPE=system WM_COMPILER=Gcc WM_MPLIB=OPENMPI 
```
如果没有报错，说明可以正式编译OpenFOAM了，先进入到Third-party下面：
```bash
./Allclean
./Allwmake -j4
#如果在集群上，可以用更多核心编译，速度会变快 srun -c64 ./Allwmake -j64
```
如果编译顺利通过，可以编译OpenFOAM:
```bash
cd ~/OpenFOAM/OpenFOAM-5.x
./Allwmake -j4 #耗时数小时
#同理如果在集群上，可以用更多核心编译，速度会变快 srun -c64 ./Allwmake -j64
```
如果没有报错，可以执行一下blockMesh，如果看到出现OpenFOAM的标志，就是安装成功了
![image.png](https://cdn.nlark.com/yuque/0/2022/png/34567141/1669996035908-f8cdf2e6-bd70-412c-9b9c-6f5b489cc556.png#averageHue=%23fcfcfc&clientId=u417a18bd-49e7-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=346&id=u284e3e3e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=346&originWidth=914&originalType=binary&ratio=1&rotation=0&showTitle=false&size=150713&status=done&style=none&taskId=u75ff3184-01f4-48ed-a678-871b005d833&title=&width=914)

# 可能遇到的错误
目前我遇到的错误，绝大多数情况都是找不到mpi.h，这个是由于openmpi没有配置好。如果按照文中的步骤，最后能看到mpi的环境成功配置应该就没有问题。
其次，找不到****.h或者***.c大部分是因为缺少依赖库。例如我一开始在本地虚拟机安装了glibc，libtool库，但是集群中没有，编译一直不过，这个时候建议首先检查依赖库。
最后一个，如果Third-party能够成功编译，基本上OpenFOAM也能顺利编译，如果最后实在是编译不过，也找不到原因。可以将本地编译好的Thirdparty打包上传到集群再解压，然后再重新编译，这也是为什么要保持本地虚拟机和集群，操作系统以及安装操作一致性的原因。因为本地虚拟机，权限和上网可控，但是集群都不行，本地的编译出现问题较容易解决。
最后看一下我自己的.bashrc文件
![image.png](https://cdn.nlark.com/yuque/0/2022/png/34567141/1669996433706-a2323a2a-3a2f-4a42-aac0-f72f887a5100.png#averageHue=%23f6f9e9&clientId=u417a18bd-49e7-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=235&id=u51a4108e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=235&originWidth=480&originalType=binary&ratio=1&rotation=0&showTitle=false&size=161333&status=done&style=none&taskId=u101bf169-95f6-4f9e-8754-ba3cd834169&title=&width=480)



