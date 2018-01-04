# ws215i-rootfs
本项目包含ws215i的rootfs和系统烧录U盘的制作工具。



开发者也可以修改本项目代码自行制作：

1. 定制的ws215i rootfs和烧录U盘镜像
2. 自己定制的ws215i系统U盘（即操作系统运行在U盘上，而不是mmc上）



本项目以Ubuntu Base 16.04.3 LTS版本为基础，该rootfs压缩包已经包含在本项目中。



## Quick Start

```bash
# 中华人民共和国境内开发者，执行下面3个命令时建议使用VPN
git clone https://github.com/wisnuc/ws215i-rootfs
npm i
node prepare-wisnuc.js --all

# 中华人民共和国境内开发者，执行下面3个命令时建议关闭VPN
sudo ./build-rootfs-emmc-base.sh
sudo ./build-burn-base.sh
sudo ./build-image.sh

# 输出文件位于output目录下
```



## 合成过程

`wisnuc.js`脚本文件用于生成目标系统上的预部署目录`/wisnuc`；

`build-rootfs-emmc-base.sh`脚本用于生成ws215i的emmc的rootfs，不包含预装的`/wisnuc`目录；

`build-burn-base.sh`脚本用于生成烧录U盘的rootfs，它是最小化的ubuntu，可以在ws215i上boot和执行自动烧录脚本。

`imagify.sh`脚本会把这些内容组合起来。首先把ws215i的base rootfs和wisnuc目录合成为`ws215i-rootfs-emmc.tar.gz`文件，它是完整的rootfs；然后imagify会使用loop device挂载imagefile，把用于usb的rootfs展开进去，把emmc rootfs的压缩文件放进去，即获得可用于烧录ws215i的U盘镜像。

图示如下：

```
prepare-wisnuc.js     build-rootfs-emmc-base.sh           build-burn-base.sh
   |                            |                             |
   v                            v                             |
output/wisnuc dir + output/ws215i-rootfs-emmc-base.tar.gz     |
                  |                                           |
                  | build-image.sh                            |
                  v                                           v
 output/ws215i-rootfs-emmc.tar.gz   +   output/ws215i-rootfs-burn-base(-debug).tar.gz
                                    |
                                    | build-image.sh
                                    v
                                (example)
  output/ws215i-ubuntu-16.04.3-node-8.9.3-appifi-1.0.11-build-180104-171627.img
```



## prepare-wisnuc.js

该脚本合成在目标系统上预部署的`/wisnuc`目录，包括：

1. 建立目录结构
2. 安装node
3. wisnuc-bootstrap
4. wisnuc-bootstrap-update
5. wetty
6. appifi

输出为当前目录的`output/wisnuc`目录。

运行该脚本无须root权限。

该脚本支持参数`--appifi-only`。

```bash
node wisnuc.js                  # 更新全部
node wisnuc.js --appifi-only    # 仅更新appifi
node wisnuc.js -a               # --appifi-only
```



## build-rootfs-emmc-base.sh

该脚本创建ws215i的rootfs (emmc)压缩包文件，但不包含预装的`/wisnuc`目录。

执行该脚本需要root权限（`sudo`）。

过程如下：

1. 创建`target/emmc`目录
2. 在目录下安装ubuntu base
3. 修改apt源为中国镜像
4. 创建如下systemd unit
   1. firstboot
   2. timesyncd
   3. wisnuc-bootstrap-update，包括timer和service
   4. wisnuc-bootstrap
   5. wetty
   6. wired.network
   7. resolv.conf，先放入一个临时版本，在chroot最后更新其为生产环境版本
   8. hosts
   9. hostname
5. chroot
   1. 安装deb包
   2. 创建wisnuc用户，加入sudo
   3. 安装ws215i内核并阻止内核升级
   4. 使能所有需要的systemd服务，禁用samba和minidlna服务
   5. 创建ws215i启动需要的boot文件（包括符号链）
   6. 修改fstab
6. post-install (leave chroot)
   1. 清理内核deb文件
   2. 清理apt
   3. 更新resolv.conf (symlink to systemd resolv.conf)
7. 最后把rootfs打包成`output/ws215i-rootfs-emmc-base.tar.gz`



## build-burn-base.sh

该脚本生成USB烧录盘的最小文件系统，USB烧录盘本身也是一个包含完整rootfs的ubuntu运行系统，不只是ramdisk。其内容和emmc镜像相仿，做了裁剪。

该脚本需要root权限（`sudo`）。

该脚本接受参数`--debug`，debug模式下最终输出的镜像文件会包含openssh server，方便开发者调试。

```bash
sudo ./build-burn-base.sh            # 生成output/ws215i-rootfs-burn-base.tar.gz
sudo ./build-burn-base.sh --debug    # 生成output/ws215i-rootfs-burn-base-debug.tar.gz
```



## build-image.sh

该脚本合成上述内容：

1. 先把emmc rootfs (base)和预置的`/wisnuc`目录合成成为`output/ws215i-rootfs-emmc.tar.gz`压缩包；
2. 创建一个临时文件，用loop device挂载，然后创建分区和ext4文件系统；
3. 展开burn rootfs (base)到目标文件系统上；
4. 装入合成的压缩包；
5. 生成最终的镜像文件。

该脚本需要root权限。

该脚本支持`--debug`参数。如果提供该参数，会使用debug版本的burn rootfs。

输出文件的命名规则如下：


```bash
# 非debug版本
ws215i-ubuntu-16.04.3-node-${Node版本}-appifi-${Appifi版本}-build-${时间戳}.img

# debug版本
ws215i-ubuntu-16.04.3-node-${Node版本}-appifi-${Appifi版本}-build-${时间戳}-debug.img
```

其中时间戳格式为`yymmdd-HHMMSS`，例如`180104-171627`。



## 文件

`assets`目录 下包含：

1. ws215i的内核包（debian格式），内核版本为4.3.3
2. Ubuntu Base压缩包
3. apt的sources.list文件，使用中国源





## 其他问题

1. 该制作过程使用了chroot，mount，loop device等功能，无法在绝大多数云主机上运行；
2. 注意chroot环境下resolv.conf的配置；
3. 因为有chroot和装包过程，所以systemd官方的firstboot服务无法使用，我们自己定义了一个firstboot service；


遇到任何问题请在本项目中提交issue，谢谢。



lewis.ma#winsuntech.cn

2018-01-04


