---
layout:     post
title:      "A207实验室服务器使用指南"
subtitle:   ""
date:       2017-06-19
author:     "SmartYi"
header-img: "img/in-post/server-guideline/head.jpg"
tags:

---

## A207实验室Amax服务器使用指南

> Created  By MingkuanLi At 2017-04-13

### 服务器信息

|               服务器IP地址                |              CPU               |    GPU     |  内存   |         磁盘         |
| :----------------------------------: | :----------------------------: | :--------: | :---: | :----------------: |
| `192.168.0.21` | Intel Xeon E5-2620 v3 / 2.4GHz | Tesla K40c | 128GB | 512GB SSD / 4T HDD |
| `192.168.0.70` | Intel Core i7-6700 / 3.40GHz | GeForce GTX 1080 Ti | 48GB | 512GB SSD / 4T HDD |

### 概览

- 服务器主要使用LXC容器进行用户间的管理与隔离
- LXC可以简单理解成一个虚拟机，但没有实际进行虚拟化，而是在Linux内核对每个用户进行了隔离
- 每个人对自己的LXC有完全的控制权限（超级管理员权限），但没有权限访问别人的LXC以及宿主机，这样的优点在于对LXC的操作（例如重启）不会影响其他用户，也不会影响宿主机。
- LXC的图形界面使用VNC进行远程连接
- 宿主机的资源（硬盘、内存、CPU、GPU等）是所有LXC用户共享的
- 分配在`192.168.0.21`的发行版是Ubuntu 14.04 LTS 64bit，桌面系统使用XFCE4，NVIDIA显卡驱动版本为375.26
- 分配在`192.168.0.70`的发行版是Ubuntu 14.04 LTS 64bit，桌面系统使用Gnome，NVIDIA显卡驱动版本为381.22

### 创建新用户

1. 创建虚拟机前联系一下`ken4000kl@gmail.com`或`mingkuan.li@foxmail.com`

2. 访问`<server-ip>:8123`，点击注册输入用户名和密码，这个密码为虚拟机超级管理员用户登录密码以及VNC远程桌面的密码

3. 点击创建虚拟机后，耐心等待，此过程会持续3分钟左右。

4. 创建成功后，回到Web主页面，可以看到用户名对应的虚拟机处于STOPPED状态

   ![1](/img/in-post/server-guideline/1.png)

5. 点击最左侧的开始按钮启动虚拟机，等待后虚拟机状态变成RUNNING代表启动成功，点击暂停按钮后状态变成STOPPED后代表关闭成功，第三个按钮为**删除虚拟机**（点击后没有确认！慎重！）

6. **注意：**每个用户分配了10个端口，具体的端口映射功能还暂未实现，其中第一个端口例如limkuan虚拟机下对应的9280端口为SSH端口，第二个端口9281端口为VNC远程桌面的端口。

### 管理虚拟机

##### 1. SSH远程连接

```sh
ssh -p <port> root@222.200.185.76
```

把\<port\>换成对应的第一个端口号，然后输入注册时的密码后就可以登录到当前用户的虚拟机的终端内，注意登录进去是超级管理员root账号

##### 2. VNC远程连接

- 使用VNC进行远程连接，需要下载VNC Viewer

- VNC Viewer是跨平台的，Windows/Linux/Mac/iOS/Android/Chrome都有对应的客户端

- 前往[VNC Viewer官网](https://www.realvnc.com/download/viewer/)下载对应平台的客户端

- （以Chrome应用为例）在Address里输入Amax服务器的IP地址以及分配到的第二个端口号

  ![2](/img/in-post/server-guideline/2.png)

- 接下来输入在注册过程中填入的密码，即可登录进入桌面系统

##### 3. SFTP文件传输

- 使用任何一个支持SFTP协议的软件（例如Filezilla、winscp等）

- （以Filezilla为例）在主机框里输入sftp://\<ip\>，用户名输入root，密码为注册时填的密码，端口号为分配的第一个端口号，点快速连接后即可看到虚拟机内的文件列表，即可进行上传下载等文件管理

  ![3](/img/in-post/server-guideline/3.png)

### 更多配置（Optional）

##### 1. 修复Xfce桌面部分问题

- **TAB按键失灵：**这个问题可能是由于Xfce的按键冲突引起的，需要到应用程序菜单→设置→窗口管理器→键盘→切换同一应用程序窗口，然后点击清除按钮清除Super+制表的快捷键绑定即可

- **终端字符重叠：**这个问题是字体导致，更换另一种字体可以解决这个问题。具体设置的位置在编辑→终端首选项→外观→字体，选择Ubuntu Mono或其他自己即可解决问题

- **用户目录下中文目录名**：可以手动把通过改系统语言把用户主目录下的文件夹改成英文名方便输入

  ```bash
  # 暂时更改系统语言为英文
  export LANG="en_US.UTF-8"
  # 修改用户主目录下的文件夹名称
  xdg-user-dirs-gtk-update
  ```

##### 2. 更改VNC远程桌面分辨

- SSH登录进LXC内，后续命令如果是root账户那么不需要输入`sudo`

- 修改VNC配置文件`sudo nano /etc/init.d/vncserver`

- 找到GEOMETRY项，如下图所示

  ![4](/img/in-post/server-guideline/4.png)

- 把GEOMETRY修改成自己想要的分辨率，然后点击Ctrl+O保存，Ctrl+X退出。

- 重启VNC服务器`sudo service vncserver restart`后重新连接即可修改VNC的分辨率。

  ![5](/img/in-post/server-guideline/5.png)

##### 3. 更换软件源

- 官方的软件源比较慢，可以使用例如中科大或清华大学在教育网内的软件源，可以在后续安装软件过程中获得比较好的下载速度
- 具体配置方法参考[Ubuntu官方文档](http://wiki.ubuntu.org.cn/模板:14.04source)

##### 4. 安装其他软件

- 常用软件安装包位于LXC容器的`/mnt/UserData/Public`，包括如下：
  - `Sublime Text-build3126`
  - `PyCharm-2017.1.1`
  - `Matlab R2016b`
  - `Intellij Idea-2017.1.1`
  - `CLion-2017.1.1`
  - `Android Studio-v2.3`
  - `NVIDIA驱动375.26`
  - `CUDA7.5`
  - `CuDnn-v6.0`
  - `OpenCV-3.0.0`
  - `OpenBLAS`
- 有同学下载了其他软件或新版本的软件也欢迎更新进去

##### 5. 安装CUDA

> 以下的教程为总结我成功Caffe框架以及cuda的历史记录总结得出，可能有遗漏的地方，如果发现什么问题请联系`mingkuan.li@foxmail.com`补充修改

- 复制安装文件到虚拟机内，需要到的文件有：

  - CUDA离线安装包：`cuda_8.0.61_375.26_linux.run` （推荐使用run的离线安装包，使用deb会使用软件源中的NVIDIA驱动而导致一起问题）
  - CUDNN安装包：`cudnn-7.5-linux-x64-v6.0.tgz` （找到相应版本的CUDNN安装包）
  - NVIDIA驱动：`NVIDIA-Linux-x86_64-381.22.run` （找到对应显卡的驱动）

- 安装CUDA：在终端中执行`cuda_8.0.61_375.26_linux.run`，注意的是选择不安装显卡驱动。

- 安装CuDNN

  ```shell
  # 解压cudnn-7.5-linux-x64-v6.0.tgz到cuda文件夹
  tar xvf cudnn-7.5-linux-x64-v6.0.tgz
  cd cuda
  sudo cp lib64/lib* /usr/local/cuda/lib64
  sudo cp include/cudnn.h /usr/local/cuda/include
  cd /usr/local/cuda/lib64
  sudo chmod +r libcudnn.so.6.0.20
  sudo ln -sf libcudnn.so.6.0.20 libcudnn.so.6
  sudo ln -sf libcudnn.so.6 libcudnn.so
  sudo ldconfig
  ```

- 重新安装NVIDIA驱动 **（使用deb安装的时候需要做）**

  ```bash
  sh NVIDIA-Linux-x86_64-375.26.run --no-kernel-module
  ```

- 测试CUDA

  - 输入`nvidia-smi -L`能正确显示出GPU的信息

    ![6](/img/in-post/server-guideline/6.png)

  - 编译cuda给的例子，然后使用deviceQuery测试

    ````shell
    cd /usr/local/cuda/samles
    sudo make all -j24
    cd bin/x86_64/linux/release
    ./deviceQuery
    ````

    输入结果最后为Result=PASS即安装成功

    ![7](/img/in-post/server-guideline/7.png)

##### 6. 配置Caffe框架

- 复制安装文件到虚拟机内，需要到的文件有：

  - `caffe-master.zip`
  - `OpenBLAS-develop.zip`
  - `opencv-3.0.0.zip`
  - `opencv_contrib-3.0.0.zip`

- 安装依赖库

  ```shell
  sudo apt-get install libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev python-dev python-numpy libtbb-dev libjpeg-dev libpng12-dev libtiff4-dev libjasper-dev libdc1394-22-dev libgphoto2-dev libavresample-dev python-pip openjdk-7-doc openjdk-7-jdk openjdk-7-jre openjdk-7-source cmake 
  sudo apt-get install libprotobuf-dev libleveldb-dev libsnapper-dev libopencv-dev libboost-all-dev libhdf5-serial-dev libgflags-dev libgoogle-glog-dev liblmdb-dev protobuf-compiler libsnappy-dev lib32z1 lib32ncurses5 lib32bz2-1.0 lib32stdc++6 gfortran
  ```

- 编译安装OpenBLAS

  ```shell
  # 解压OpenBLAS-develop.zip
  unzip OpenBLAS-develop.zip
  cd OpenBLAS-develop
  make
  sudo make PREFIX=/usr/local install
  ```

- 编译安装OPENCV

  ```shell
  # 解压opencv-3.0.0.zip以及opencv_contrib-3.0.0.zip
  unzip opencv-3.0.0.zip 
  unzip opencv_contrib-3.0.0.zip

  cp -r opencv_contrib-3.0.0 opencv-3.0.0/contrib
  cd opencv-3.0.0
  mkdir build
  cd build
  #后面EXTRA_MODULE_PATH为contrib的路径，可以使用绝对路径
  #别忘了最后的两个点
  cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/usr/local -D OPENCV_EXTRA_MODULES_PATH=../contrib/modules/ ..
  make -j24
  sudo make install
  ```

- 测试OpenCV是否安装成功

   ```shell
   #当前位于opencv-3.0.0目录内
   cd samples/python2
   python asift.py
   #运行结果
   ```

   ![8](/img/in-post/server-guideline/8.png)


- 编译安装Caffe

   ```shell
   #解压caffe-master.zip
   unzip caffe-master.zip
   # 安装Caffe的依赖
   for req in $(cat requirements.txt); do sudo pip install $req; done
   #复制编译的配置文件
   cp Makefile.config.example Makefile.config
   ```

- 编辑并保存`Makefile.config`

  ```cmake
  # 第5行，取消注释表示使用CUDNN
  USE_CUDNN := 1

  # 第21行，取消注释，使用OPENCV 3
  OPENCV_VERSION := 3

  # 第28行，指定cuda的位置
  CUDA_DIR := /usr/local/cuda

  # 第36行，注释compute_60以及compute_61的三行
  CUDA_ARCH := -gencode arch=compute_20,code=sm_20 \
  		-gencode arch=compute_20,code=sm_21 \
  		-gencode arch=compute_30,code=sm_30 \
  		-gencode arch=compute_35,code=sm_35 \
  		-gencode arch=compute_50,code=sm_50 \
  		-gencode arch=compute_52,code=sm_52 \
  		#-gencode arch=compute_60,code=sm_60 \
  		#-gencode arch=compute_61,code=sm_61 \
  		#-gencode arch=compute_61,code=compute_61

  # 第50行，修改为open表示使用OpenBLAS
  BLAS := open

  # 第63行，如果安装了MATLAB，那么在这里制定MATLAB的路径
  MATLAB_DIR := /usr/local/MATLAB/R2016b
  ```

- 编译Caffe

  ```shell
  make all -j24
  make test -j24
  make runtest
  #如果runtest结果为pass那么编译成功
  ```

- 编译Matlab wrapper（前提是正确安装了Matlab）

  ```shell
  make matcaffe
  make mattest
  ```

### 已知问题

对于下面的问题，如果有相应的解决方法，请联系`mingkuan.li@foxmail.com`

1. 不能弹出认证框输入密码
2. Sublime Text 3无法输入中文

