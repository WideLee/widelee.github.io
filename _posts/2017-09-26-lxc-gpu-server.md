---
layout:     post
title:      "实验室配置公用服务器过程"
subtitle:   ""
date:       2017-09-26
author:     "SmartYi"
header-img: "img/in-post/lxc-gpu-server/head.jpg"
tags:
    - Linux
---

## 实验室配置公用服务器过程

> Created  By MingkuanLi On 2017-09-26

### 1. 概述

曾经实验室服务器由于不规范的使用出现过太多奇奇怪怪的问题，可能某个同学辛辛苦苦配好的环境不久之后另外一个同学配了另一个环境就把原来的环境弄坏了的事情也时有发生，大约半年前看到了[这个](https://zhuanlan.zhihu.com/p/25710517)，于是打算好好管理实验室的服务器资源，提高大家的学习研究效率。

一开始我们听说了[Linux Container](https://linuxcontainers.org/)可以做用户环境隔离，而且对性能影响不大，可以把GPU映射进去进行加速深度学习训练网络之类的任务，于是开始针对LXC开始配置用户环境的隔离。后来后来才搞明白了docker的用法，发现docker封装了更多基本的操作例如备份恢复之类的，而且有NVIDIA官方维护的nvidia-docker对GPU映射兼容性可能更好，然而之前针对LXC做的工作太多，以后有机会再想办法全面迁移到docker。

感谢一开始kenlee一起来折腾这个东西

### 2. 安装显卡驱动以及CUDA

- 参考[这个博客](https://wizyoung.github.io/Ubuntu%E4%B8%8BGTX1080%E6%98%BE%E5%8D%A1%E9%A9%B1%E5%8A%A8%E6%8A%98%E8%85%BE%E5%B0%8F%E8%AE%B0/)进行配置
- 安装显卡驱动首先要禁用NVIDIA开源的驱动nouveau
- 新建文件`/etc/modprobe.d/blacklist-nouveau.conf`然后加入如下内容：

```sh
blacklist nouveau
blacklist lbm-nouveau
options nouveau modeset=0
alias nouveau off
alias lbm-nouveau off
```

- 然后在终端执行

```sh
echo options nouveau modeset=0 | sudo tee -a /etc/modprobe.d/nouveau-kms.conf
sudo update-initramfs -u
```

- 重启电脑后即可禁用开源驱动nouveau
- 接下来去nvidia驱动官网下载对应显卡的驱动一般为xxxx.run
- 按`Ctrl+Alt+F1`进入tty1，登录后停止lightdm/gdm

```sh
sudo service lightdm stop
```

- 接下来安装nvidia驱动，**注意GPU是否有图形输出**，例如Tesla K40c属于服务器级的GPU，不带VGA或者HDMI显示输出的，在安装的时候需要指定--no-opengl-files的参数

```sh
# GPU 没有图形输出
sudo ./NVIDIA-Linux-x86_64-352.99.run --no-opengl-files
# GPU 有图形输出
sudo ./NVIDIA-Linux-x86_64-352.99.run
```

- 重启后可以进入图形界面使用nvidia-smi验证NVIDIA驱动是否安装成功

![验证NVIDIA驱动](/img/in-post/lxc-gpu-server/1.png)

- 接下来安装CUDA，在官网下载相应版本CUDA，建议选择run文件
- 安装build-essential

```sh
sudo apt-get install build-essential
```

- 执行`./cuda-xxxx.run`，根据提示，选择不安装NVIDIA驱动，然后其他默认就好
- 装完后进入`samples`目录使用`make`编译，编译结果在`bin`文件夹里，执行`deviceQuery`验证CUDA是否安装成功

### 3. 安装配置LXC

- 参考LXC官网的[Getting Started](https://linuxcontainers.org/lxc/getting-started/)
- 使用`apt-get`安装`lxc`

```sh
sudo apt-get install lxc
```

- **坑**，我使用Ubuntu 14.04.5默认安装的lxc版本为1.0.10，根据Getting Started配置完id_map以及`/etc/lxc/lxc-usernet `后，发现普通用户的`unprivileged containers`启动不起来，错误日志说创建cgroups失败，而使用sudo创建的containers能启动成功

  - 曾经打算全部使用sudo启动管理，但后来映射GPU设备到Container内的时候，出现如下的错误

```
Failed to initialize NVML: Unknown Error
```

  - Google后[发现这个](https://superuser.com/questions/1017194/no-cuda-capable-device-is-detected-inside-lxc-container)，怀疑是映射到Container内的`/dev/nvidia0`等设备文件权限的问题，**暂时未能解决**
  - 后来搜到了这个[github issue](https://github.com/lxc/lxc/issues/1489)，问题和我们的基本类似，在`lxc-start`的时候会报如下错误：

```
% lxc-start -n app --logfile /tmp/lxc.log --logpriority DEBUG
lxc-start: cgmanager.c: lxc_cgmanager_create: 301 call to cgmanager_create_sync failed: invalid request
lxc-start: cgmanager.c: lxc_cgmanager_create: 303 Failed to create pids:lxc/app
lxc-start: cgmanager.c: cgm_create: 650 Error creating cgroup pids:lxc/app
lxc-start: start.c: lxc_spawn: 910 failed creating cgroups
lxc-start: start.c: __lxc_start: 1149 failed to spawn 'app'
lxc-start: lxc_start.c: main: 341 The container failed to start.
lxc-start: lxc_start.c: main: 345 Additional information can be obtained by setting the --logfile and --logpriority options.
```

  - 参考LXC项目的owner [hallyn](https://github.com/hallyn) 的解答，定位出问题可能是LXC里面的cgroup管理认不到`pids`这个cgroup，而`pids`这个cgroup是Linux内核4.3之后添加进去的（[这里](http://man7.org/linux/man-pages/man7/cgroups.7.html)）

```
pids (since Linux 4.3; CONFIG_CGROUP_PIDS)
              This controller permits limiting the number of process that
              may be created in a cgroup (and its descendants).

              Further information can be found in the kernel source file
              Documentation/cgroup-v1/pids.txt.
```

  - 而Ubuntu 14.04.5的Linux内核是4.4，这就是为什么另外一台服务器是Ubuntu 14.04.4的系统然后安装上lxc后没出现奇怪的问题，而新装的这台服务器`lxc-start`启动不起来 
  - `pids`这个cgroup是管理`subsystem`的进程数量的，这个我们暂时不需要，于是考虑在Ubuntu启动的时候，把这个功能禁用掉
  - 打开`/boot/grub/grub.cfg`，找到对应的启动参数添加`cgroup_disable=pids`，添加如下：

```sh
menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-89259414-c487-4d5e-a614-1cdc6b76113a' {
  recordfail
  load_video
  gfxmode $linux_gfx_mode
  insmod gzio
  insmod part_msdos
  insmod xfs
  set root='hd0,msdos1'
  if [ x$feature_platform_search_hint = xy ]; then
    search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos1 --hint-efi=hd0,msdos1 --hint-baremetal=ahci0,msdos1  89259414-c487-4d5e-a614-1cdc6b76113a
  else
    search --no-floppy --fs-uuid --set=root 89259414-c487-4d5e-a614-1cdc6b76113a
  fi
  linux /boot/vmlinuz-4.4.0-96-generic.efi.signed root=UUID=89259414-c487-4d5e-a614-1cdc6b76113a ro  quiet splash $vt_handoff cgroup_disable=pids
  initrd  /boot/initrd.img-4.4.0-96-generic
}
```

  - 重启系统后，还需要给`.local`以及`.local/share`执行的权限，理论上lxc就能正常启动了

```sh
chmod +x ~/.local
chmod +x ~/.local/share
lxc-start -n template -d
```

### 4. 配置LXC内的各种服务

- 这里是打算把实验室的主页以及各种服务（例如Git，SVN，FTP等）都放在虚拟机里，方便备份迁移，尽量做到宿主机崩了之后重装系统可以快速恢复里面的服务
- 在这个过程中遇到的主要问题是让服务在后台自启动，大概有如下几个方式
  - 配置`/etc/init/xxxx.conf`（未测试，但`vsftpd`好像是通过这种方式自启动的）
  - 修改`/etc/rc.local`，然后`lxc-start`的时候指定命令`cmd`
  - 启动后，另外通过`lxc-attach`启动相关的服务，通过`nohup`实现后台运行

### 5. 配置提供用户使用的LXC模板

- 修改LXC配置文件config映射NVIDIA硬件到虚拟机内

```
lxc.mount.entry = /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry = /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry = /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
```

- 虚拟机内安装NVIDIA驱动，由于Container是和宿主机共享内核的，因此需要加上`--no-kernel-module`参数

```sh
sudo ./NVIDIA-Linux-x86_64-352.99.run --no-kernel-module
```

- 和宿主机一样安装cuda
- `sudo apt-get install ubuntu-desktop`
- 安装vnc4server
- 配置`.vnc/xstartup`

```sh
#!/bin/sh

# Uncomment the following two lines for normal desktop:
# unset SESSION_MANAGER
# exec /etc/X11/xinit/xinitrc

[ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
xsetroot -solid grey
# vncconfig -iconic &
vncconfig -nowin &
# x-terminal-emulator -geometry 80x24+10+10 -ls -title "$VNCDESKTOP Desktop" &
# x-window-manager &

export XKL_XMODMAP_DISABLE=1
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS

ibus-daemon -d -x

export GTK_IM_MODULE=ibus
export QT_IM_MODULE=ibus
export XMODIFIERS=@im=ibus

gnome-panel &
gnome-settings-daemon &
metacity &
nautilus -n &
gnome-terminal &

```

- 配置vnc的自启动脚本`/etc/init.d/vncserver`

```sh
#!/bin/sh -e
### BEGIN INIT INFO
# Provides:          vncserver
# Required-Start:    networking
# Default-Start:     3 4 5
# Default-Stop:      0 6
### END INIT INFO
PATH="$PATH:/usr/X11R6/bin/"

# The Username:Group that will run VNC
export USER="root"
#${RUNAS}

# The display that VNC will use
DISPLAY="1"

# Color depth (between 8 and 32)
DEPTH="24"

# The Desktop geometry to use.
#GEOMETRY="<WIDTH>x<HEIGHT>"
GEOMETRY="1920x1080"

# The name that the VNC Desktop will have.
NAME="vnc-server"

OPTIONS="-name ${NAME} -depth ${DEPTH} -geometry ${GEOMETRY} :${DISPLAY}"

. /lib/lsb/init-functions

case "$1" in
start)
log_action_begin_msg "Starting vncserver for user '${USER}' on localhost:${DISPLAY}"
su ${USER} -c "/usr/bin/vncserver ${OPTIONS}"
;;

stop)
log_action_begin_msg "Stoping vncserver for user '${USER}' on localhost:${DISPLAY}"
su ${USER} -c "/usr/bin/vncserver -kill :${DISPLAY}"
;;

restart)
$0 stop
$0 start
;;
esac

exit 0
```

### 6. 宿主机到LXC内的端口映射

- 主要使用iptables进行端口映射

```sh
# 添加把宿主机的10000端口转发到LXC的10.0.3.205的5901端口规则
sudo iptables -t nat -A PREROUTING -p tcp --dport 10000 -j DNAT --to-destination 10.0.3.205:5901
sudo iptables -t nat -A POSTROUTING -p tcp -d 10.0.3.205 --dport 10000 -j MASQUERADE

# 删除把宿主机的10000端口转发到LXC的10.0.3.205的5901端口规则
sudo iptables -t nat -D PREROUTING -p tcp --dport 10000 -j DNAT --to-destination 10.0.3.205:5901
sudo iptables -t nat -D POSTROUTING -p tcp -d 10.0.3.205 --dport 10000 -j MASQUERADE

# 列出当前所有的nat端口转发规则
sudo iptables -t nat --list
```

- **坑**， 当时通过上述规则映射了宿主机的80端口到虚拟机A的80端口，导致其他虚拟机上不了网，所有访问外网的80端口的都会被转发到虚拟机A里面

  - 这个问题大概是因为添加`PREROUTING`的时候，指定了所有设备里进入的访问80端口数据包都进行转发，而从其他虚拟机里出来的数据包时流经虚拟设备lxcbr0的，也匹配了上面那个规则，导致被转发了
  - 解决的方法是在添加转发规则的时候通过参数`-i`指定端口，只有外网的端口才进行转发

```sh
sudo iptables -t nat -A PREROUTING -p tcp -i eth0 --dport 10000 -j DNAT --to-destination 10.0.3.205:5901
```

  ​