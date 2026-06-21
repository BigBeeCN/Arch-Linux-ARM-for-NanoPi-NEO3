# 快速上手
如果你不想要进行任何编译操作，也可以直接使用我编译好的系统镜像。
***
***适配NanoPi NEO3的Arch Linux ARM SD卡启动镜像下载***

[直链下载](https://open.666ma.fun:8443/share/ArchLinuxARM-6.1.63-aarch64-NanoPi-NEO3-sd-20260615.img)

[百度网盘](https://pan.baidu.com/s/1Lx47It868f6pIKMhsfFqhw?pwd=406p)

[夸克网盘](https://pan.quark.cn/s/9aafaea72d8c)
***
下载完成后你会得到一个`.img`文件，你可以使用BalenaEtcher、Rufus这类工具软件将这个系统镜像文件写入到你的SD卡中。（请直接跳转到本文的后面[[#Arch Linux ARM系统初始化]]）
# 背景介绍
## 系统选择
- FriendlyElec官方仅为NEO3这款硬件提供了有限的定制系统镜像，包含debian、ubuntu、openwrt等等，这类镜像往往内置很多我们日常使用并不需要的工具或软件，在启动时可能占用大量本来就已岌岌可危的板载RAM。
- 第三方社区例如大名鼎鼎的Armbian官网也仅有Debian 13(trixie)的Minimal (CLI)定制镜像（最新构建日期：2026年6月13日）。
- NEO3这款硬件采用的CPU虽是我们熟知的瑞芯微RK3328，但是NEO这个产品线因为单网口的设计，不能完美满足主流的软路由需求，所以远不如R系列热门，同样采用了RK3328的NanoPi R2S在网络上就能轻松找到各种各样的系统镜像。官方对NEO3的态度也很明显，最新的镜像上传日期还停留在2023年5月23日（截至2026年6月1日）。
- Arch Linux ARM(ALARM)继承了Arch Linux滚动更新、高度可定制和极简的特点，并且相关社区活跃度较高，配合NEO3等等这类ARM开发板，可玩性非常高。
## 编译方式
### 官方闭源
在NanoPi NEO3的FriendlyElec官方Wiki中，提到了两种编译方式：
1. 利用官方的`sd-fuse`脚本库一键打包出`.img`镜像（🌟🌟🌟最推荐）
    克隆`sd-fuse_rk3328`仓库，并下载一个在官方脚本支持列表里的固件包，让脚本在本地编译出针对NEO3的U-Boot和Kernel后，再将你想要最终编译出的系统对应的固件包里的`rootfs.img`替换掉之前下载的官方支持固件包里的`rootfs.img`。这种方式最简单且过程中不易出现兼容性等各种问题，成功率较高，个人最推荐。
2. 通过命令行手动编译并分区写入SD卡（🌟推荐，可以尝试）
    按照官方wiki中提到的命令手动进行`make`操作完成编译，并手动给SD卡挂载和分区，再将你前面编译好的各个`.img`刷入对应的扇区或分区。这种方式也不是很复杂，每一步基本上官方Wiki里都有介绍，但是毕竟编译的系统不在官方的支持列表里，过程中可能会遇到工具硬件等的兼容性问题，可以尝试。
### 社区开源
得益于开源社区的强大，RK3328这颗芯片在现代的U-Boot和Linux Kernel主线中已经得到了极其完善的支持。因此产生了纯开源的主线化（Mainline）编译方案：
- 直接向Linux Torvalds和U-Boot官方主线代码索取资源完成编译（⚠️困难，一般不推荐）
    编译纯开源的安全固件（TF-A）和主线U-Boot（包含纯开源内存初始化），使用主线Kernel（内核），再手动构建根文件系统，从而完成ARM启动链的完整拼图。这种方式流程极为复杂，且与硬件厂商设计的底层存在兼容问题，启动失败率较高，需要一定的技术经验积累和面对各种报错和异常的痛苦承受能力；但是在折腾结束后，你可以随时拉取Linux主线的最新代码，支持极高强度的加密验签，所以建议享受推演系统底层逻辑、且具备极客精神的硬核开发者进行尝试。
# 编译教程
## 环境准备
下面在Ubuntu 26.04系统中进行操作
1. 安装必要的编译工具🛠
    ```bash
    sudo apt update
    sudo apt install -y build-essential gcc-aarch64-linux-gnu curl git libncurses5-dev libssl-dev mtools bc dosfstools python3 e2fsprogs fdisk android-sdk-libsparse-utils exfatprogs
    
    #创建软链接骗过官方脚本，让后续编译过程中实际运行的是系统本地的原生编译器
    sudo ln -s /usr/bin/aarch64-linux-gnu-gcc /usr/local/bin/aarch64-linux-gcc
    sudo ln -s /usr/bin/aarch64-linux-gnu-g++ /usr/local/bin/aarch64-linux-g++
    ```
2. 获取打包脚本与基础模板
    这里选用较新的6.1内核，并下载官方的`Ubuntu Noble`作为打包“基底”
    ```bash
    git clone https://github.com/friendlyarm/sd-fuse_rk3328.git -b kernel-6.1.y
    cd sd-fuse_rk3328
    wget http://112.124.9.243/dvdfiles/rk3328/images-for-eflasher/ubuntu-noble-core-arm64-images.tgz
    tar xvzf ubuntu-noble-core-arm64-images.tgz
    ```
## 开始编译
接下来的操作都需要确保位于上一步生成的`sd-fuse_rk3328`目录下
1. 为NEO3编译专属的Kernel
    - 先替换官方脚本里写死的一些老旧依赖包名
        ```bash
        find . -name "*.sh" -exec sed -i 's/android-tools-fsutils/android-sdk-libsparse-utils/g' {} +
        find . -name "*.sh" -exec sed -i 's/exfat-utils/exfatprogs/g' {} +
        ```
    - 下载并编译6.1内核
        ```bash
        git clone https://github.com/friendlyarm/kernel-rockchip --depth 1 -b nanopi-r2-v6.1.y kernel-rk3328
        KERNEL_SRC=$PWD/kernel-rk3328 ./build-kernel.sh ubuntu-noble-core-arm64
        ```
        **⚠️注意：这一步编译过程可能需要数十分钟，具体时长取决于编译设备CPU性能，请耐心等待直到出现成功提示。**
    等待内核编译跑完后，生成相应的内核驱动已经存放在`out/output_rk3328_kmodules/lib/modules`目录下了
2. 准备好Arch Linux的根文件
    仍然保持在`sd-fuse_rk3328`目录下，新建空目录并下载解压Arch Linux ARM官方的根文件系统
    ```bash
    mkdir arch_rootfs
    
    #中国大陆地区建议将官方源替换为清华源https://mirrors.tuna.tsinghua.edu.cn/archlinuxarm/os/ArchLinuxARM-aarch64-latest.tar.gz
    wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
    
    #这一步解压必须使用sudo以保留Linux系统的核心文件权限
    sudo bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C arch_rootfs/
    ```
3. 注入NEO3专属驱动
    ```bash
    sudo cp -r out/output_rk3328_kmodules/lib/modules/* arch_rootfs/lib/modules/
    ```
4. 打包与生成烧录镜像
    利用FriendlyElec官方提供的打包生成脚本来实现一键打包与生成`.img`文件
    ```bash
    sudo ./build-rootfs-img.sh arch_rootfs ubuntu-noble-core-arm64
    ./mk-sd-image.sh ubuntu-noble-core-arm64
    ```
    等待最后一条命令跑完后，SD卡烧录镜像（rk3328-sd-ubuntu-noble-core-6.1-arm64-yyyymmdd.img）就保存在`sd-fuse_rk3328/out/`目录下
# 后续操作
## 提取出SD卡烧录镜像文件
前面我们已经在Ubuntu系统里安装了`python3`，因此我们可以直接在存放镜像文件的目录里搭建一个临时的网页服务器，然后在同网段网络环境中的浏览器里访问它，直接把文件下载下来
1. 在Ubuntu系统中获取IP地址
    在终端输入以下命令
    ```bash
    ip a
    ```
    在输出结果中找到它的局域网IP（通常是`192.168.x.x`）
2. 启动临时下载服务
    进入存放镜像文件的目录，并启动Python服务器
    ```bash
    cd ~/sd-fuse_rk3328/out
    python3 -m http.server 8000
    ```
    当看到终端卡住显示类似`Serving HTTP on 0.0.0.0 port 8000...`，则说明服务器已经启动
3. 在浏览器中下载
    打开同网段下设备的浏览器，在地址栏输入：
    `http://<你刚才查看的Ubuntu系统IP>:8000`
    页面上会列出`~/sd-fuse_rk3328/out`目录下的所有文件，直接点击`rk3328-sd-ubuntu-noble-core-6.1-arm64-yyyymmdd.img`文件，就会开始下载
# Arch Linux ARM系统初始化
## 初始账户
### 普通用户
- 账号：`alarm`
- 密码：`alarm`
### 超级管理员
- 账号：`root`
- 密码：`root`
## 系统初始化
下面的操作进行前请确保你已成功登入并拿到root权限
### 关闭Pacman沙盒
我们使用的定制内核默认没有没有开启`Landlock`这个安全特性，会触发Pacman下载的安全阻断，因此需要提前关闭Pacman的沙盒功能
1. 编辑Pacman配置文件
    ```bash
    nano /etc/pacman.conf
    ```
2. 添加DisableSandbox选项
    在配置文件中找到`[options]`这一区块，找到`DisableSandboxFilesystem`和`DisableSandboxSyscalls`这两行，删掉它们前面的`#`（注释符），让它们生效
    修改后：
    ```conf
    DisableSandboxFilesystem
    DisableSandboxSyscalls
    ```
### 初始化Pacman密钥环
由于这是一个全新系统，必须先初始化`pacman`包管理器的密钥环，否则后续安装任何软件都会报签名错误。请依次执行：
```bash
pacman-key --init
pacman-key --populate archlinuxarm
```
完成初始化后，你就可以执行`pacman -Syu`来更新系统了。
