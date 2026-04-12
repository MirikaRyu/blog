+++
date = '2026-04-06T17:10:04+08:00'
title = 'Xiaomi AX3000T AP 固件精简与配置：自定义 UBoot 与主线 OpenWrt'

tags = ['Linux', 'OpenWrt', 'AP', 'Uboot', 'Network', 'Embeded', 'Customize']
categories = ['Linux']
image = 'cover.webp'
+++

## 前言
最近正在搭建一台5G CPE，因此需要找个 AP 来发射 WiFi 信号。最开始使用的方案是在软路由里面一个空闲的 M.2 M-Key 插槽上面插入一张
转接 MT7922 进行射频，来来去去换了几次天线，倒腾了一个星期后发现信号质量甚至不如手机热点 😅。毕竟受限于发射功率
（只有可怜的 3 dBm）只能使用 2.4 GHz 的频段，再加上只有 2X2 MIMO 以及不支持 WiFi-6 BeamForming 等技术的 debuff，
这种情况也算是在意料之中了。

所以就在各种机缘巧合之下发现了这个 AX3000T，双频 WiFi-6 ~反正 WiFi-7 6 GHz 在国内也别想用~ 和5根天线、完美的 OpenWrt 主线支持、
相对简洁便携的外观、以及最重要的不到100元的价格。唯一美中不足的它的这4个网口全是只有千兆的。诶，但是，查阅相关资料
可以发现其 SoC 和交换机之间的链路是有 2.5 Gbps 的，这也就意味着我们可以通过链路聚合实现并发时突破千兆的速度（有坑，往下看）。

## 准备工作
作为一名（自认为）合格的赛博洁癖患者，自己编译干净的固件自然是必不可少的一环。大体的环境准备工作可以
参照[官方文档](https://openwrt.org/docs/guide-developer/toolchain/buildsystem_essentials)进行，编译时如果遇到其它
缺少依赖项的问题再根据自己的环境灵活变通即可。

还有就是由于需要对 Uboot 进行自定义，因此 Hard Brick 的风险大大增加了 😇，最好还可以准备一条串口转 USB 的线以防万一。

>**观前提示** ⚠️<br>
>这**不是**一篇完全新手向的文章，并且我也不能确定其中的所有操作是否正确无误。如果你不能理解在刷入过程中的每一步
>操作意味着什么，请**立即停止**并严格按照主线 OpenWrt 的[官方文档](https://openwrt.org/inbox/toh/xiaomi/ax3000t)进行操作。
>本人不对任何 data loss 或者 bricking 负责。

## 刷入 OpenWrt 主线固件
诶，等等，不是要自己编译固件吗？怎么直接刷主线固件了？这是因为主线固件的工具和环境相比我们的精简版完善得多，可以先在主线版本中
确认自己的硬件信息、对总体的网络配置进行验证、以及在自定义固件出现奇怪行为的时候进行对照，总之，
保留一份主线固件备用还是很有必要的。

然后就可以按照[文档](https://openwrt.org/inbox/toh/xiaomi/ax3000t#flash_instructions)的指引开 SSH，**备份相关分区**，
然后刷入 Uboot 和 OpenWrt。如果发现没法使用 `apk` 下载 `kmod-mtd-rw` 这个包，可以到以下 URL 寻找并手动下载这个包：
>https://downloads.openwrt.org/releases/<Distro 版本>/targets/mediatek/filogic/kmods/<内核版本>/

用 `scp` 上传到路由器之后使用以下命令安装：
```shell
apk add --allow-untrusted --force-non-repository ./<你的文件名>
```

这里再次强调一下一定要备份 **Factory** 这个分区，其它的分区如果不考虑换回原厂固件其实可以不太关注，但这个分区存放的是
WiFi 校准数据，一机一份，丢了的话就喜提一台4口千兆交换机。

---

进入新的 OpenWrt 系统后我们可以使用这行命令来查看自己的交换机芯片：
```shell
dmesg | grep -e an8855 -e mtk_soc_eth
```
我的运气比较好，抽中的是 RD03v1 的版本，因此显示的是：
>[    2.208970] an8855-switch an8855-switch.2.auto: probe with driver an8855-switch failed with error -110<br>
>[    2.224236] mtk_soc_eth 15100000.ethernet eth0: mediatek frame engine at 0xffffffc081580000, irq 75

然后使用 `cat /proc/mtd` 来查看一下主线 ubootmod 版本的默认分区表：
```
dev:    size   erasesize  name
mtd0: 00100000 00020000 "BL2"
mtd1: 00040000 00020000 "Nvram"
mtd2: 00040000 00020000 "Bdata"
mtd3: 00200000 00020000 "Factory"
mtd4: 00200000 00020000 "FIP"
mtd5: 00040000 00020000 "crash"
mtd6: 00040000 00020000 "crash_log"
mtd7: 00040000 00020000 "KF"
mtd8: 07000000 00020000 "ubi"
```
其中的 `BL2`, `Factory` 以及 `FIP` 可以说是三个最重要的分区。`BL2` 存放的是负责进行 DDR 等基础硬件初始化的
Trusted Boot Firmware，`FIP` (Firmware Image Package) 用于存放 Arm BL31 和 BL33 UBoot 固件。至于其它几个分区，
`crash`, `crash_log` 和 `KF` 分区 dump 出来之后查看其内容均为全空，`Nvram` 用于存放原厂 UBoot 固件的环境变量，`Bdata` 里面
好像是一堆乱七八糟的密钥以及默认的 SSID/MAC/SN，如果不恢复原厂就可以不用理会。

---

这里有必要简单说一下这类硬路由设备在 OpenWrt 上的典型存储层次。

先从最下层的存储设备开始，这类设备一般使用的是廉价的 NOR/NAND Flash，并使用 SPI 总线与 SoC 连接。因此这类芯片表现出来的行为
更像是只能顺序读写的字符设备而不是普通的块设备。除此之外，NAND Flash 在进行写入的时候要先将整个「擦除块」全部擦除后才能进行写入，
而同时这类芯片写入寿命也不高，同一个块擦写次数过多就会导致该块永久损坏。

接着在内核中对于这类设备就有了第一个抽象层，MTD (Memory Technology Device)，如果在安装好的 OpenWrt 中执行 `ls /dev/mtd*` 就能够
看到这些设备。这一层仅仅只是对设备进行的简单封装，使得用户空间程序能够使用简单、统一的 syscall 来读写存储芯片。其中 `mtdX` 就是
第 X 个可读写的 MTD 分区的字符设备接口，`mtdXro` 对应的就是只读接口，而 `mtdblockX` 则是内核驱动在对应分区上虚拟出的块设备接口，
能够让该分区使用块设备的接口进行读写。

至此，对于静态数据或者不会频繁写入的数据来说，已经能够直接通过上面这个 MTD 接口进行交互了，例如 Bootloader，Nvram 等分区。但
对于需要频繁写入的数据来说直接使用此接口会造成严重的问题，~你也不想用一小会就报废了吧~。所以，我们需要通过 UBI
(Unsorted Block Images) 这个管理层进行控制。UBI 很像 LVM，但其主要用途是为了进行磨损均衡，即维护一个逻辑擦除块到物理擦除块的
映射，使得每次写入都能尽可能的分摊到每个块上。UBI 也使用分卷进行管理，每个分卷和 MTD 的表现类似，提供字符设备和可选的块设备接口。

>**提示** ℹ️<br>
>到后面你可能会发现清除配置文件是通过直接移除再创建 `rootfs_data` 这个 UBI 卷来实现的，这个过程是在 UBoot 中完成的，且并没有将
>其格式化为 UBIFS 文件系统。但为何系统仍然可以正常启动？难道是 UBI 创建分卷的时候会自动格式化？
>
>如果我们在清除完数据之后再启动系统，查看 `dmesg` 可以发现其实是内核在启动的时候帮我们完成了这一步：
> >[    8.239123] UBIFS (ubi0:1): default file-system created

<br>
再向上就是我们相对熟悉的一般文件系统了，这里主要使用 UBIFS 和 SquashFS/EROFS。这里需要单独把 UBIFS 给拎出来，因为只有它是基于
字符设备接口的文件系统，而其余两个都是基于块设备的。而至于具体的文件系统有什么特点此处不再展开，查阅资料即可。

更多信息可以查看[文档](https://openwrt.org/docs/techref/flash.layout)。

---

安装完主线固件后可以尝试一下 UBoot 的 TFTP 启动方式，后面我们很可能会频繁地使用这个方法来加载临时系统。

先将路由器下电，然后戳着 Reset 按钮再插上电源，等待指示灯变成蓝色后就可以松开了。将上位机的 IP 设置为 `192.168.1.254` 并确保
和路由器之间的网络可达。接着找到路由器需要的恢复镜像文件，
主线固件默认为 `openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-initramfs-recovery.itb`。将工作目录切换到该文件夹下，
并运行一个 TFTP Server，这里推荐直接使用 `dnsmasq`：
```shell
sudo dnsmasq -i <连接路由器的链路名称> -d --port=0 --enable-tftp --tftp-root=$(pwd)
```
由于需要绑定的为特权端口 `69`, 因此需要 Root 权限。使用 `-d` 开启 debug 模式，`--port=0` 禁用 DNS 功能，剩下的两个选项用于开启
TFTP Server。

当你能够在输出中看到 "Send xxx" 的时候就说明 UBoot 已经获取到镜像文件并启动了。

## 配置自定义 OpenWrt 固件
我们先从 Github Mirror 拉取源码并进行 menuconfig：
```shell
git clone https://github.com/openwrt/openwrt
cd openwrt
make menuconfig
```
由于我们的目标是配置出一个专为 AP 服务的精简固件，因此不需要去执行 `./scripts/feeds update -a` 之类的操作，使用默认软件包即可。

然后就可以开始配置了，下文所有的配置都会放到[我的仓库](https://github.com/MirikaRyu/netcfg)中，可以使用 `git apply` 直接
应用这些 patches。

### 目标配置
前3个 Target System, Subtarget, Target Profile 分别选择 MediaTek ARM, Filogic 8x0, Xiaomi AX3000T (U-Boot layout) 即可。

接下来在 Target Images 选项中可以看到以下选项：
- **ramdisk**：是否构建 initramfs 镜像，关闭后不会生成 initramfs-* 镜像，不建议关闭。
- **Root fs archives**：将编译出的 rootfs 进行归档方便查看。
- **Root fs images**：选择镜像中的 rootfs 文件系统。EXT4 是可写的，不建议选。在剩下两个中据说某为的 EROFS 性能更好，因此这里
选择 EROFS。
- **Image Options**：配置根分区大小和选择是否持久化 `/var` 目录。

### 构建选项
进入 Global build settings，里面绝大部分选项都不需要修改。

找到 Gernel build options 下的 Kernel build options，里面控制了内核相关的功能和编译选项，可根据具体需求更改。
接下来把 Package build options 中的第 2,3,4 项都勾上，这三个选项分别是 `-ffuntion-sections --gc-sections`, `-flto` 以及使用号称
速度最快的链接器（如果你发现有别的比它快，可以去提交 bug report 🤣）。不过需要注意的是可能有极少数包在开启 LTO 后会出问题，
不过我还没有遇到过，因此先保持开启。

下面的加固选项 Hardening build options 也可根据自己的需求调整，具体每个选项是什么意思可以去询问 LLM，总体上每个等级开得越高，
性能开销也会越大。在这种更看重安全性的场景中开高一些应该无所谓。

---

然后是 Advanced configuration options。

找到 `ccache`，好东西，编译缓存，反复编译的时候能加加速。
向下翻到 Target Options，里面有一个 Target Optimizations 的选项，里面默认开的是 `-Os` 即优化占用空间，我们将它改成 `-O2`，
优化运行速度。当然我也不太确定开启后是否会造成不良影响，总之 "It works on my machine"。

---

最后这个 Image configuration 开启后可以设置一些系统基本信息，可按需配置。里面 Seperate feed repositories 中的选项基本用不上，
可以全关了。

### Base system / 基本工具
绝大多数选项的说明都非常清晰，作为 AP 的话，相比默认配置，可以移除 `dnsmasq`, `firewall` 这两个包。

然后如果想进一步精简，可以打开 Customize busybox options，删除里面不需要的工具：
- **Archival Utilities**：这边就是一堆压缩格式，喜欢啥选啥。
- **Coreutils**：不要删太猛，有一次把 `date` 给删除了，结果发现 `sysupgrade` 依赖这个来校验时间。
- **Debian Utilities**：这里我只保留了一个 `which`。
- **Editor**：有编辑配置文件的需要，因此 `vi` 相关的选项基本拉满了。
- **Networking Utilities**：这里我只保留了 `IPv6 support`, `netstat`, `ntpd`, `ping/ping6` 以及 `udhcpc/udhcpc6`，
其它 `ifconfig/route` 之类的功能将会被 `iproute2` (Network -> Routing and Redirection -> `ip-full`) 这个包完全替代。
- **System Logging Utilities**：中的 `logger` 同样被 `sysupgrade` 依赖，不能删除。

如果你发现某个很重要的功能在这里面没有开启，那就说明肯定是有其它的替代，不需要再去勾选；如果你需要某个工具的完整版本，
可以去 Network/Utilities 之类的大分类下去找找，找到后再回来取消勾选这里的即可。

这里面也有几个不能删的选项，比如 `uci`，用于简化使用脚本进行系统配置，初始配置生成依赖这玩意，删除之后会导致没有默认 IP
从而无法使用 SSH 连接。至于 `apk-mbedtls` 和 `ca-bundle`，如果确定不再需要通过包管理器安装额外的软件包就可以直接扬了。

接下来 AX3000T 作为 AP 还需要**添加** `bridger` 这个软件包用于从 WLAN 到 LAN 的 WED (Wireless Ethernet Dispatch)。
根据文档的说法，这个功能可以将无线流量直接卸载到硬件进行转发从而缓解 CPU 压力。

但是，还记得开头说的使用链路聚合的坑吗？

如果你像我一样打算使用链路聚合的话这个包和 WED 功能就没有什么用处了，实际测试下来开启与否 CPU 都会被吃满。根据 LLM 的说法，
像链路聚合这种 packet 必须要经过内核处理的协议，是无法直接通过硬件进行转发的。

其它信息可以进一步查看具体的[Wiki](https://openwrt.org/docs/guide-user/network/wifi/wed)页面。
>**提示** ℹ️<br>
>这个包需要 LLVM 的 eBPF 工具链，从头编译的时候需要编译整个 LLVM 🙃。<br>
>有人向 Kernel 提了 Patch 来在内核中实现这个包的功能，不过目前貌似中道崩殂了？反正 6.12 的这个版本应该是依然需要这个包的。

### Network / 网络工具
在下面那一堆未分类的选项中只需要保留作为依赖项的 `iw` 以及 `uclient-fetch`。由于不需要进行 WAN 拨号，因此可以移除 `ppp` 这个包。

然后在 Routing and Redirection 中选择前述的 `ip-full`。

最后进入 WirelessAPD 这个分类，其中包含控制 WiFi 认证的相关工具。如果不需要让这个路由器去连接其它 WiFi 的话，这里只需要选择
`hostapd-basic-mbedtls` 即可。下面的 `wpad-*` 其实是 `hostapd` + `wpa-supplicant`，选了之后会多一个 RADIUS 服务器。

### Kernel modules / 内核模块
这里面绝大部分的功能都用不上，在关闭了 `ppp`, `firewall` 等功能之后，可以在这里把一些无用的内核模块也关闭掉。
像是 Netfilter Extensions 下的模块可以全部删除，Network Support 中只需要开启链路聚合必要的 `kmod-bonding` 以及被 `bridger` 包
依赖的几个 `kmod-sched-*` 即可。

除此之外，还可以使用 `make kernel_menuconfig` 来单独调整内核的配置，不过似乎用处不太大，没有太多可以调整的。

### 其它配置
剩余的选项的描述应该也非常清晰了，大多数都是作为依赖项出现被强制选择的。有几点需要说明的如下：
- Boot Loaders 中只选择第一个，即包含 "mt7981,ddr3" 的那个即可。
- Utilities -> Boot Loaders 中有一个 `uboot-envtools`，由于后面要折腾 UBoot，没有这个基本寸步难行，不建议删除。
 但由于这个包会创建 `/etc/config/ubootenv` 这个意义不明的文件，要是真想删除可以在调试完 UBoot 后再删。
- Libraries 下有一个 `libustream-mbedtls` 这个包，好像是用来给包管理器做 HTTPS 认证的，如果前面保留了包管理器这里也要保留这个。

---

在以上配置工作完成之后就可以先使用以下命令试试能否正常编译：
```shell
make -j$(nproc) V=s
```
如果嫌输出太乱可以去掉 `V=s` 等编译失败的时候再加上。

## 配置自定义 UBoot 固件
我们自定义 UBoot 的目的主要为开启 Netconsole，规范分区以及自定义启动行为。

### 开启 Netconsole
在刚见到这玩意的时候还以为是串口的完全上位替代品，怎么用怎么舒服，直到某次重启之后发现 UBoot 卡死了... 😵

尽管如此，我依旧认为这是在路由器这类难以直接连接 tty 的设备上很重要的一个功能，它能够让你在不用拆机连接串口的情况下获得一个 tty。
但不知为何网上的对这个功能讨论度低的可怜，就连 OpenWrt 的文档中都没怎么提到 🤔。

我们先来看看主线 UBoot 的 Netconsole 如何开启。

小米 AX3000T 默认的主线 UBoot 其实已经在编译选项中开启了对该功能的支持：`CONFIG_NETCONSOLE=y`，因此我们只需要配置以下几个环境
变量即可使用，可以在 OpenWrt 中使用 `fw_setenv` 进行设置：
- **ipaddr**：本机 IP，上位机连接的时候需要的 IP，这里就按照默认值 `192.168.1.1`，不过要注意不要和主路由的 IP 冲突。
- **ncip**：目标主机的 IP，也就是使用 Netconsole 机器的 IP，这里和 OpenWrt 默认值保持一致，设为 `192.168.1.254`。
- **stdin**：标准输入，这里必须设置为 `nc`，LLM 可能会让你加上 `serial`，但这里不行，原因稍后揭晓。
- **stdout**：标准输出，同上，必须设为 `nc`。
- **stderr**：标准错误输出，同上，必须设为 `nc`。

接下来我们就可以在上位机连接 Netconsole 了。首先确保本机 IP 已经设为上面的 `ncip` 且和路由器之间是可达的，
然后可以使用 `dl` 目录下 UBoot 源码中的 `tools/netconsole` 这个脚本连接，注意该脚本依赖于 `openbsd-netcat` 或者是 `socat`：
```shell
./netconsole 192.168.1.1
```
如果上面的脚本出现了奇怪的问题的话可以试试下面这个简化版本（依赖于 `openbsd-netcat`），注意按 \<Ctrl-c\> 会直接退出：
```shell
#! /usr/bin/env sh

stty -icanon -echo
nc -u -l -p 6666 -b
stty sane
```
如果不出意外的话，重启路由器后应该能够在终端中出现 OpenWrt 的启动选择界面，按 \<Esc\> 即可退出至 UBoot 命令行。

---

接下来看看在我们的自定义固件中要如何配置。

首先有必要来说一下这玩意非常离谱的一点：UBoot 的 console 初始化是串行且阻塞的。这也就意味着如果出于任何原因，Netconsole 没有
初始化成功，那 UBoot 就会一直卡在这个地方。换句话说，如果一直启用 Netconsole，则每次开机时在局域网内必须有一个 IP 为 `ncip`
的设备，否则 UBoot 不会启动 🫪。这个问题我们在下面配置启动流程的时候再解决。

然后我们来修复 stdio 只能使用一个 console 的问题。只需要在 Uboot 的 config 中加入以下几行配置：
```
CONFIG_CONSOLE_MUX=y
CONFIG_SYS_CONSOLE_IS_IN_ENV=y
```
第一个配置开启了 UBoot 的终端多路复用功能，使得可以同时处理多个来源的输入/输出，只有这样才能将 stdio 的 3 个值设为
`serial,nc`（并没有接串口验证是否真的可以 multiplex 🤪）。第二个则是显式指明了可以在环境变量中设置输入输出流。
~虽然默认配置好像就可以~

更多 UBoot Netconsole 的用法可以查阅[文档](https://docs.u-boot.org/en/latest/usage/netconsole.html)。

---

现在你可能又会遇到一个问题：这些配置文件在哪里，我要怎么修改？

这些机器特定的 UBoot 配置均以补丁文件的形式存放在 `packages/boot/uboot-mediatek/patches` 目录下，像这台 AX3000T 的补丁文件名就叫
`440-add-xiaomi_mi-router-ax3000t.patch`。一般来说，修改这些文件要使用 OpenWrt 专门处理 patches 的机制，即 Quilt。

但是，虽然是个昏招，显然不如直接修改补丁文件要来的直接。我们这里对整个项目的改动并不大，也不用考虑提交给上游，因此可以直接修改
补丁文件然后祈祷上游不要对这个文件再做什么改动 (~又不是不能用~)。就是修改的时候要比较小心，需要遵守 patch 文件的格式。如果构建时
报错 "Failed to apply patch" 之类的，就回来检查一下格式。Patch 文件的具体格式询问 LLM 即可。

### 规范默认分区
我们的目标是将整个系统的分区表变成下面这样：
```
dev:    size   erasesize  name
mtd0: 00100000 00020000 "BL2"
mtd1: 00080000 00020000 "Env"
mtd2: 00200000 00020000 "Factory"
mtd3: 00200000 00020000 "FIP"
mtd4: 07120000 00020000 "Sys"
```
上面这个是在 OpwnWrt 上使用 `cat /proc/mtd` 输出的。需要注意的是像这种使用 MTD 的嵌入式设备的分区方式，不同于常见的桌面设备
使用 GPT 这类分区表，是通过写死在设备树里的配置划分的。而 UBoot 和 Kernel 又各自使用一份设备树，也就是 UBoot 和内核看到的
分区表和设备其实是可以不一样的。因此如果我们想统一整个系统的分区表，就要分别修改 UBoot 和内核的设备树。

我们在上面提到的 patch 中翻找一下，应该能够找到在 `spi_nand@0` 下的 `partitions`，其中的一个 entry 长成下面这样：
```dts
partition@0 {
    label = "bl2";
    reg = <0x00 0x100000>;
};
```
具体含义如下：
```
partition@形式起始地址 {
    label = "显示的标签";
    reg = <真实起始地址 分区大小>;
}
```
后面内核设备树的定义方式也是类似的。

然后我们就可以给分区改名，或者将分区进行合并了。注意要计算好合并后的大小和偏移量以及不要去动那3个分区即可。

我这边对分区名进行了更改，将 `Nvram` 和 `Bdata` 分区合并为 `Env` 分区，同时将 `crash`, `crash_log`, `KF` 和 `ubi` 分区合并为了
一个大的 `Sys` 分区：
```dts
partition@0 {
    label = "BL2";
    reg = <0x00 0x100000>;
};

partition@100000 {
    label = "Env";
    reg = <0x100000 0x80000>;
};

partition@180000 {
    label = "Factory";
    reg = <0x180000 0x200000>;
};

partition@380000 {
    label = "FIP";
    reg = <0x380000 0x200000>;
};

partition@580000 {
    label = "Sys";
    reg = <0x580000 0x7120000>;
};
```

### 自定义启动行为
UBoot 在启动时执行的各种逻辑是通过设置环境变量来实现的，而主线固件的逻辑让强迫症有点抓狂了 😡。下面就来重新实现自己的逻辑。

首先配置一些参数：
```conf
ipaddr=192.168.1.1
ncip=192.168.1.254
serverip=192.168.1.254

led_rec=blue:status
led_pwr=yellow:status
console=earlycon=uart8250,mmio32,0x11002000 console=ttyS0

loadaddr=0x46000000
bootconf=config-1
bootfile=recovery.itb
bootfile_upd=sysupgrade.itb
bootargs=console=ttyS0,115200n8 console_msg_format=syslog
```
这里 Netconsole 的相关参数仅设置 `ncip` 即可，不要动 stdio 的三个变量，其余配置保持原来的默认值即可。我这里还修改了 `bootfile`
和 `bootfile_upd`，这两个变量代表了在后面使用 TFTP 从服务器获取相关镜像文件时的文件名。

接下来只保留我们需要的菜单功能，即默认启动、TFTP Recovery、TFTP Production 和恢复出厂：
```conf
bootdelay=2
bootmenu_delay=2
bootmenu_default=0
bootmenu_0=Default Boot=run bootcmd
bootmenu_1=TFTP Recovery Boot=run boot_tftp_recovery
bootmenu_2=TFTP Production Boot=run boot_tftp_production
bootmenu_3=Reset Factory Settings=run reset_factory; reset
bootmenu_4=Reboot=reset
```
这里没有直接设置标题的原因是 `$ver` 这个变量必须被动态替换才能生效，因此需要专门的逻辑来实现。

然后先实现两个 TFTP Boot：
```conf
boot_tftp_production=tftpboot $loadaddr $bootfile_upd; bootm $loadaddr#$bootconf
boot_tftp_recovery=led $led_rec on; while true; do tftpboot $loadaddr $bootfile && bootm $loadaddr#$bootconf; sleep 1; done
```
逻辑都比较简单，先使用 `tftpboot` 加载镜像到指定的内存然后直接 `bootm`。对于 `boot_tftp_recovery` 来说就是再多打开一个
恢复指示灯并死循环进行 TFTP Boot。

现在来看看如何从 UBI 文件系统中启动 OpenWrt：
```conf
ubi_name=Sys
ubi_vol_fit_name=fit
ubi_vol_dat_name=rootfs_data
ubi_read_fit=ubi part $ubi_name; ubi read $loadaddr $ubi_vol_fit_name && iminfo $loadaddr
ubi_init_dat=ubi part $ubi_name; if ubi check $ubi_vol_dat_name; then else ubi create $ubi_vol_dat_name - dynamic; fi
ubi_boot=if run ubi_read_fit && run ubi_init_dat; then bootm $loadaddr#$bootconf; fi
```
由于我们将原本的 `ubi` 改成了 `Sys`，因此这里要手动指定名称，但分卷的名称依然与主线一致。当执行 `ubi_boot` 的时候，先选中分区，
然后从 `fit` 分卷加载镜像并进行校验，成功后再检查 `rootfs_data` 分卷，没有则创建，如果能够成功完成上述步骤就使用 `bootm` 启动。

---

接下来处理环境变量持久化和恢复出厂的问题。

主线 UBoot 将环境变量放在 `ubootenv` 和 `ubootenv2` 分卷当中，好处是有冗余备份和磨损均衡。我的选择是将环境变量放在合并后
的 `Env` 分区中，有 512KiB 的空间，即4个擦除块的大小。这个大小是不用考虑 UBI 了，只能使用 Raw MTD，不过虽说 NAND Flash 不耐擦，
但对于这种配置完成后基本不动的机器来说10万+的擦写次数也够用好多年了。

将 UBoot Env 放在 MTD 分区中还需要对原来的 config 做出一些更改，将原来 config 中的 ubi/redundent 的配置替换为如下配置：
```conf
CONFIG_ENV_IS_IN_MTD=y
CONFIG_ENV_MTD_DEV="Env"
CONFIG_ENV_SIZE=0x60000
CONFIG_ENV_OFFSET=0x0
```
这几行指定了环境变量放在一个叫 `Env` 的 MTD 分区上，并设定最大能够写入的大小为 0x60000 字节，写入位置距离该分区开头为 0x0。
由于紧接着的就是 `Factory` 分区，因此这里留下了一个擦除块大小的 Guard Area。

环境变量问题主要是如何实现 First Boot，即第一次启动或者恢复出厂后自动写入，而其余时候自动加载已保存的变量：
```conf
env_auto_init=if test "$env_inited" = "1"; then else setenv env_inited 1; saveenv; fi
```
其实也很简单，定义一个 Flag，首次启动的时候初始化后保存，再次启动的时候如果探测到这个标志则跳过写入即可。主线的实现方式是使用
一个自修改变量进行标记，本质是一样的，就是会更绕一些。

恢复出厂就直接将存放环境变量的 `Env` 分区和系统配置的 `rootfs_data` 分卷擦除即可：
```conf
mtd_env_name=Env
reset_factory=mtd erase $mtd_env_name; ubi part $ubi_name; ubi remove $ubi_vol_dat_name
```

---

现在还要确保 Netconsole 不会在启动时把整个 UBoot 给卡住。

由于 Netconsole 使用的是 UDP，因此我首先考虑的是能否将 `ncip` 设置为广播地址从而规避找不到设备的问题。从结果上看，找不到设备的
问题似乎解决了，但现在只能在上位机收到 Netconsole 的 stdout 却无法回传 stdin。

然后尝试的做法是看看 UBoot 中有没有一种机制可以在网络连接失败后自动跳过 Netconsole 的初始化，找半天就找到一个 `netretry=no`，
但是显然也是无效的。

所以现在剩下的方法只有在每次启动时判断是否开启 Netconsole。但 UBoot 里好像也没有什么靠谱的检测手段，`ping` 一个不存在的地址必须
等待 ARP 失败计数耗尽才返回，那现在就只能依靠用户来手动指定了。

这边我选择的是 Opt-out，即默认开启，需要手动设置 `no_nc=1` 来关闭 Netconsole，配置如下：
```conf
check_nc=if test "$no_nc" = "1"; then else setenv stdin serial,nc; setenv stdout serial,nc; setenv stderr serial,nc; fi
```
注意这行代码**必须**在自动环境变量保存后才能执行，否则会将临时设置的 stdio 变量持久化。

---

最后再来实现按键、灯光和 Preboot 的逻辑。

按键检测比较简单，逻辑如下：
```conf
check_mesh=if button mesh; then run boot_tftp_recovery; fi
check_reset=if button reset; then led $led_rec on; run reset_factory; reset; fi
set_title=setenv bootmenu_title "    [0;34m( ( ( [1;39mOpenWrt[0;34m ) ) )     [33m${ver}[0m"
```
主线的 Mesh 按钮是没有定义功能的，但如果每次想启动 Recovery Ramdisk 都要去戳 Reset 按钮好像有点太蠢了，所以我这里将 Mesh 按钮
定义为直接进行 TFTP Recovery Boot，而 Reset 按钮则是进行恢复出厂。动态设置 `bootmenu_title` 的逻辑也放在了这里。

但如果把以上的逻辑放在 `bootcmd` 中执行的话那需要在 `bootmenu_delay` 结束后才会被触发，因此这里需要用到 UBoot 的 Preboot 功能，
使其能够在 UBoot 刚初始化完成，在 `boot_delay` 之前就执行逻辑。该功能在主线没有开启，需要在 config 下加入这一行：
```conf
CONFIG_USE_PREBOOT=y
```
然后就可以定义 `preboot` 环境变量：
```conf
preboot=led $led_pwr on; run check_reset; run check_mesh; run env_auto_init; run set_title; run check_nc
```
以及一个 `bootcmd`：
```conf
bootcmd=run ubi_boot; run boot_tftp_recovery
```

这里和主线固件不一样的是在恢复模式下路由器的指示灯会变成紫色 🫨。

>**注意** ⚠️<br>
>永远确保任何一个启动路径最后都有个 Fallback 兜底，或者是永远能够拿到 UBoot Shell，这样只要 UBoot 没死都能救得回来。

## 配置自定义内核设备树
内核这边的修改相比 UBoot 就要轻松多了，这边主要修改几个网口的名称以及分区表。

在 `target/linux/mediatek/dts` 这个目录下应该能够找到 `mt7981b-xiaomi-mi-router-common.dtsi`
和 `mt7981b-xiaomi-mi-router-ax3000t-ubootmod.dts`，我们只需要修改这两个设备树文件。

### 修改网口名称
先打开带 "common" 的那个文件，直接在里面搜索 "wan"，应该会出来两个地方，分别对应联发科的 MT7531AE 以及 AN8855 两种交换机，
可以都修改了，找到每个 `port` 中的 `label`，改成喜欢的名字即可。不过两个交换机中都各有一个和其它有点不同的 `port`，我猜测这个
是交换机连接 SoC 的端口，就不需要修改了。

### 修改内核分区表
操作和 UBoot 的大同小异，依旧是先打开带 "common" 的设备树文件，找到 `spi_nand`，该合并的合并，该删除的删除，不要动那3个分区。
然后打开另一个 "ubootmod" 的设备树文件，计算好 `Sys` 分区的起始地址和偏移量再修改即可。

唯一需要注意的是内核设备树这边可以添加 `read-only;` 只读属性，添加了这个属性的分区如果想被写入需要加载 `mtd-rw` 这个内核模块。

## 编译新固件并刷入
完成以上步骤之后就可以进行完整的构建了，构建产物在 `bin/targets/mediatek/filogic` 这个目录下。

如果你不放心编译出来的固件是否正常，可以拿 `openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-preloader.bin` 这个文件
和下载的官方 ubootmod 的版本对比一下是否一致（`xxd` 是 `vim` 的一部分，可以使用 `tinyxxd` 替代）：
```shell
xxd openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-preloader.bin > /tmp/my.hex
xxd openwrt-25.12.2-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-preloader.bin > /tmp/official.hex
diff /tmp/my.hex /tmp/official.hex
```
如果以上的 diff 结果一致或者只有编译时间和文件尾部的 32B 的校验值不一样的话那就说明编译过程基本没出什么问题。

---

接下来就要刷入我们编译好的固件了。

>**警告 ⚠️：刷写 UBoot 的过程中出现错误或者 UBoot 本身有问题可能会导致必须使用串口或者编程器修复**

先通过 UBoot 的 TFTP 启动**主线**的 initramfs recovery 镜像，和刷主线 UBoot 类似，先加载 `mtd-rw`，然后再上传自己的镜像并使用
以下命令刷入：
```shell
mtd write /tmp/openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-bl31-uboot.fip FIP
```
然后务必校验刷入是否成功：
```shell
mtd verify /tmp/openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-bl31-uboot.fip FIP
```
BL2 preloader 就没有刷写的必要性了，毕竟前面已经比对过内容其实是一样的。

确认刷写成功后重启路由器，如果没出什么岔子现在在 UBoot Netconsole 上应该能够看见新的 Bootmenu 了。

接下来我们使用 TFTP Boot 启动**自己编译的** initramfs recovery 镜像，并将 sysupgrade 镜像上传。然后使用以下命令格式化
新的 MTD 分区（如果你的分区表不一样请勿照搬）：
```shell
ubidetach -p /dev/mtd4
ubiformat -y /dev/mtd4
ubiattach -p /dev/mtd4
```
如果遇到在 detach 的时候遇到 "Device or resource busy" 的错误，这是可能是由于有几个奇怪的服务和内核线程占用了该 UBI 卷。直接使用
`mtd erase` 暴力抹除该分区，擦除完成后重启即可生效。但是注意这样做会丢失原来统计的擦除计数，不利于磨损均衡。

格式化完新分区后可以直接使用 `sysupgrade` 完成安装：
```shell
sysupgrade -n /tmp/openwrt-mediatek-filogic-xiaomi_mi-router-ax3000t-ubootmod-squashfs-sysupgrade.itb
```

## 配置路由器 AP 模式
### 系统整体配置
设置 Root 密码：
```shell
passwd
```

`/etc/config` 系统配置：
```ucode
config system
    option hostname '<Your hostname>'
    option timezone 'CST-8'
    option zonename 'Asia/Shanghai'
    option ttylogin '0'
    option log_size '128'
    option urandom_seed '0'
    option compat_version '1.0'

config timeserver 'ntp'
    option enabled '1'
    option enable_server '1'
    list server '0.openwrt.pool.ntp.org'
    list server '1.openwrt.pool.ntp.org'
    list server '2.openwrt.pool.ntp.org'
    list server '3.openwrt.pool.ntp.org'
```

### 网络配置
OpenWrt 中的网络配置集中在 `/etc/config/network` 这个文件中。

先定义一个 loopback interface:
```ucode
config interface 'loopback'
    option device 'lo'
    option proto 'static'
    list ipaddr '127.0.0.1/8'
```

然后进行 LACP 链路聚合，定义一个聚合设备：
```ucode
config device
    option name 'bond0'
    option type 'bonding'
    list ports 'lan2'
    list ports 'lan3'
    option policy '802.3ad'
    option lacp_rate 'fast'
    option monitor_interval '1000'
    option xmit_hash_policy 'layer3+4'
```
这里将 `lan2`, `lan3` 两个网口用于同主路由进行聚合，上面这些聚合的参数必须与主路由上设置的完全一致。

关于链路聚合需要说明的一点是，只有并发请求才能够完全占满聚合的带宽。当然 `blance-rr` 除外，但这玩意会让包乱序到达，
导致吞吐量下降，实用性不强。

接下来声明一个网桥设备，即交换机，也是这个路由器本身：
```ucode
config device
    option name 'br0'
    option type 'bridge'
    list ports 'bond0'
    list ports 'lan0'
    list ports 'lan1'
```
将聚合设备 `bond0` 以及剩余的两个 LAN 接口加入网桥，两个无线链路会在其配置文件中指定加入该网桥。

然后给这个网桥分配静态 IP 等参数：
```ucode
config interface 'lan'
    option device 'br0'
    option proto 'static'
    list ipaddr '192.168.1.2/24'
    list ip6addr 'fd11:4514:0721::2/48'
    option gateway '192.168.1.1'
    list dns '192.168.1.1'
```
### 无线配置
编辑 `/etc/config/wireless`：
```ucode
config wifi-device 'radio0'
    option type 'mac80211'
    option path 'platform/soc/18000000.wifi'
    option band '2g'
    option channel 'auto'
    option htmode 'HE40'
    option txpower '30'
    option country 'CN'

config wifi-iface 'wlan0'
    option device 'radio0'
    option network 'lan'
    option mode 'ap'
    option ssid '<Your SSID>'
    option encryption 'sae'
    option ieee80211w '2'
    option key '<Your Passwd>'
```
以上是一个 2.4 GHz 频段的配置，5 GHz 基本就是复制粘贴一份。其中具体每项配置是什么含义该如何选择
在[这里](https://openwrt.org/docs/guide-user/network/wifi/basic)已经有很详细的说明了，此处不再赘述。

最后如果要开启 WEB 功能，需要在 `/etc/modules.conf` 中手动指定模块加载参数，重启生效：
```conf
options mt7915e wed_enable=Y
```
虽然说开启了链路聚合之后 PPE 就不能加速从 WLAN 到 LAN 的流量了，但兴许 WLAN 之间的流量依然可以加速 🤔？

### UBoot 环境变量修改
由于我们修改了存放 UBoot 环境变量的位置，因此需要修改 `/etc/fw_env.config` 来指明新位置：
```
# Device    Offset    Size    EraseSize
 /dev/mtd1  0x0       0x60000 0x20000
```
默认配置文件结尾有个莫名其妙的数字，删掉后工具才能够正常工作。

在我的网络环境下，主路由已经占用了 `192.168.1.1` 这个 IP 了，这就需要将 UBoot 的 IP 进行更改以免造成冲突：
```shell
fw_setenv ipaddr 192.168.1.2
```
别忘了将 Netconsole 关掉：
```shell
fw_setenv no_nc 1
```

## 自定义 NFC 标签内容
这台小米 AX3000T 还带了一个通过 I2C 总线连接的 NFC 芯片，用于非接触连接 WiFi，如果你只是想将标签里的内容更新为自己的 SSID，直接
使用[这个项目](https://github.com/Caian/xinfc)并按照 Readme 进行操作即可。

但如果你想往这个标签里写点有意思的东西就可以继续往下看。

这个标签里的数据格式叫做 NDEF，是由 NFC 论坛搓出来的标准，涵盖了各种数据格式。我们可以在上位机使用 Python 的 `ndeflib` 这个库
实现对 NDEF 数据的编解码。比如现在我想将我的博客 URI 写到标签里，就可以用下面的 Python 脚本生成 payload：
```python
#! /usr/bin/env python

import ndef
import os

URI = "https://mirikaryu.github.io"

result = b"".join(ndef.message_encoder([ndef.UriRecord(URI)]))
result = bytes([0x03, len(result)]) + result + b"\xfe"

os.write(1, result)
```
这个库编码出的数据不带 TLV 类型标签，因此需要手动补上。如果有其它想写入的数据类型，
查看[文档](https://ndeflib.readthedocs.io/en/stable/index.html)并询问 LLM 即可。

然后我们将脚本的输出重定向到 bin 文件：
```shell
./make.py > nfc.bin
```

那现在要怎么将这个 payload 写入呢？

由于原项目没有提供写入任意 payload 的功能，因此我基于原项目的数据重新写了一个程序，能够写入至多 160 Bytes 的
payload（原项目就是这么多 🫠），超出的部分会被截断。另外由于交叉编译环境配置很麻烦，因此这个程序能够不依赖 libc 和 CRT，
理论上找一个**任意**的 AArch64 GCC/Clang 即可编译然后在路由器上使用。

先从[仓库](https://github.com/MirikaRyu/netcfg/tree/main/ap-ax3000t/tools/nfc.cpp)获取文件 `nfc.cpp`，并使用这行命令编译出
可执行程序 `nfc`：
```shell
aarch64-linux-gnu-g++ -std=c++20 -O2 -nostdlib -ffreestanding -static nfc.cpp -o nfc
```
如果你不想或者不会找新的编译器，切换到 OpenWrt 目录，使用现成的也可以：
```shell
cd tmp
../staging_dir/toolchain-aarch64_cortex-a53_gcc-14.3.0_musl/bin/aarch64-openwrt-linux-g++ -std=c++20 -nostdlib -o nfc nfc.cpp
```
上面具体 toolchain 的路径会因具体版本而不同。

接着将 `nfc.bin` 和 `nfc` 上传到路由器。执行以下命令备份原有内容 ~好像备份了也没啥用~：
```shell
./nfc dump > nfc.bak
```
>**提示** ℹ️<br>
>如果你在编译时将读写的 buffer 开得比较大，可能会发现 dump 出来的数据从 896 字节开始又有新的内容，这些内容可能与 NFC 芯片的
>配置/锁定区有关，不建议触碰。

然后将自己的 payload 写入，写入完成后可以再 dump 出来查看其内容：
```shell
./nfc write < nfc.bin
./nfc dump | hexdump
```
写入完成之后拿手机解锁后贴上去应该就能够自动打开设置的 URI 了。

---

如果你不想自己编译的话。。。。大概可能也算还有一个办法。

依旧，你得先准备好要写入的 payload，不过这次要以不带前后缀的十六进制纯文本的形式输出，每个字节之间按照空格分隔，所以你可能
要让 LLM 改改上面的那个脚本。然后在路由器上安装 `i2c-tools`：
```shell
apk add i2c-tools
```
接下来使用下面这个提示词让 LLM 帮你完成一个 Shell 脚本：
>帮我完成一个运行在小米 AX3000T OpenWrt ash 环境下的 Shell 脚本，要求如下：
>1. 从标准输入接收一个纯文本的 NFC NDEF payload，每个字节均按照空格分隔，字节不包含前后缀，总字节数不得超过 160
>2. 将读取到的字节使用 i2c-tools 中的 i2ctransfer 写入总线 0，地址为 0x57 的 NFC 芯片的 0x10 寄存器中
>3. 该设备为双字节寻址的设备，只有 i2ctransfer 能够处理这种设备

最后将生成的 Shell 脚本和 payload 上传到路由器，按照 LLM 给出的步骤进行操作，然后祈祷能够成功。

什么？你问我为什么不用后者？那是因为我写完代码之后才发现可以用 `i2ctransfer` 处理双字节寻址的设备 🤡。

## 使用体验

系统安装完成后 `fit` 分区占用 9.4 MiB，`rootfs_data` 容量为 96.9 MiB：
```
UBI version:                    1
Count of UBI devices:           1
UBI control device major/minor: 10:256
Present UBI devices:            ubi0

ubi0
Volumes count:                           2
Logical eraseblock size:                 126976 bytes, 124.0 KiB
Total amount of logical eraseblocks:     905 (114913280 bytes, 109.5 MiB)
Amount of available logical eraseblocks: 0 (0 bytes)
Maximum count of volumes                 128
Count of bad physical eraseblocks:       0
Count of reserved physical eraseblocks:  20
Current maximum erase counter value:     3
Minimum input/output unit size:          2048 bytes
Character device major/minor:            251:0
Present volumes:                         0, 1

Volume ID:   0 (on ubi0)
Type:        dynamic
Alignment:   1
Size:        78 LEBs (9904128 bytes, 9.4 MiB)
State:       OK
Name:        fit
Character device major/minor: 251:1
-----------------------------------
Volume ID:   1 (on ubi0)
Type:        dynamic
Alignment:   1
Size:        801 LEBs (101707776 bytes, 96.9 MiB)
State:       OK
Name:        rootfs_data
Character device major/minor: 251:2
```

拿手机上的 `iperf3` 在距离路由器1.5米左右处进行测速。在并发数为 4 的情况下，上传峰值有 1.6 Gbps 左右，平均在 1.2 ~ 1.5 Gbps
之间，不算稳定；下载最高只能跑到 1 Gbps 出头，平均在 900 Mbps 上下。单流上传能跑满千兆，但手机的单流下载只能跑到 450 Mbps，
拿笔记本测试单流下载可以跑满千兆。

在进行并发上传的时候两个 CPU 几乎都快吃满了，~毕竟辣鸡 A53~。

尝试过 packet steering 将中断请求进行均衡，结果吞吐量有所下降。猜测是处理 irq 的和进行包转发的不是一个核心导致 cache 频繁失效。

总的来说虽然结果不太满意，但至少把基本的带宽、稳定性以及覆盖范围的问题解决了。

## 一点吐槽
在费了几天时间折腾这玩意之后只能说 x86 生态依然是不可动摇啊，强制 UEFI + ACPI 真是比这一堆嵌入式的玩意舒服多了，不用自己做脏活，
但是谁让这些东西这么便宜呢 🤪。还有 SysV init 和 systemd 这个经典问题。尽管 SysV init 确实是简单了，但架不住别人 systemd 是
真的方便，坐等 usystemd 🤤。

说回 OpenWrt 这边，这玩意是我到目前为止用过的唯一一个 `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin` 完全分开的发行版，我想不到
有什么不把这些目录合并的理由，真是要逼死强迫症了 🙄。还有它的文档，怎么说呢 🤔。。。。。。👆🤓 诶，不如 ArchLinux！不过说实话
我也不太确定 OpenWrt 的文档水平到底如何，反正在链路聚合那里有几个参数在 OpenWrt 的文档里压根找不到 😑。
