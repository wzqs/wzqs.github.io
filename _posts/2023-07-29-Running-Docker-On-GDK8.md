---
layout: post
title: Running Docker On GDK8
category: linux
---

---
{: data-content="TL;DR"}

记录下在 GDK8(RK3328) 中出现 **Devices cgroup isn't mounted** 导致 docker 无法正常启动的解决方案，该启动异常的主要原因是因为 Docker 所需的内核选项没有开启。

参考：

https://www.nanocode.cn/wiki/docs/gdk8_primer/primer_gdk8_img

https://forum.loverpi.com/discussion/225/running-docker-on-roc-rk3328-cc

https://docs.docker.com/engine/install/troubleshoot/

---
{: data-content="解决办法"}

使用官方脚本检查下当前内核对 docker 的兼容性

```
curl https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh > check-config.sh 
bash ./check-config.sh
```

根据上面的检查结果，将出现 miss 的选项记录下来，后面会用到。

安装编译内核时会用到的软件包

```
sudo apt-get update
  sudo apt-get install bc liblz4-tool
```

  下载内核源代码(言子-聊城2版本 4.19.161-yanzi)


```
wget https://gedu.oss-cn-beijing.aliyuncs.com/Products/GDK8/Release/Ubuntu/LiaoCheng2/kernel.tar.gz

tar -xf

cd kernel
```

修改 kernel config 添加支持 Docker 所需的内核选项（参考第一步中的 miss 选项，示例仅供参考）

```
 mv arch/arm64/configs/rockchip_linux_defconfig arch/arm64/configs/rockchip_linux_defconfig.bak 
cp -r /boot/config-4.19.161-yanzi arch/arm64/configs/rockchip_linux_defconfig  
vim arch/arm64/configs/rockchip_linux_defconfig  

CONFIG_CGROUP_DEVICE=y
 CONFIG_USER_NS=y 
CONFIG_CGROUP_HUGETLB=y 
CONFIG_CFS_BANDWIDTH=y 
CONFIG_SECURITY_APPARMOR=y
 CONFIG_BRIDGE_VLAN_FILTERING=y
```

  在内核目录下编译 

```
touch .scmversion 
make ARCH=arm64 rockchip_linux_defconfig 
make ARCH=arm64 gdk850.img -j8 LOCALVERSION=-yanzi
```

<img width="963" alt="Pasted Graphic" src="https://github.com/wzqs/wzqs.github.io/assets/71961807/6ea2c933-9509-4533-aeae-b95ee5c859ce">

查看 kernel 对应的 devname (仅供参考，不要搞错）  

```
fdisk -l

udevadm info -n /dev/mmcblk2p4

# 它俩的 magic number 应该是一样的，这里会显示以 `ANDROID!` 开头

sudo od -c /dev/mmcblk2p4 | more
sudo od -c boot.img | more
```

<img width="798" alt="image" src="https://github.com/wzqs/wzqs.github.io/assets/71961807/d838f1a4-b7dd-450f-9a29-3efa158599b7">


  烧录 kernel  

```
dd if=boot.img of=/dev/mmcblk2p4 seek=0
sync
reboot
```

  重新安装 docker

  ```
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo docker run hello-world
  ```

成功运行

<img width="966" alt="Pasted Graphic 2" src="https://github.com/wzqs/wzqs.github.io/assets/71961807/aa6f7e52-1e24-454b-b953-32844d8332ea">

---
{: data-content="闲扯"}

arm64 的软件生态没有想象中那么差，功耗也低，随便跑点什么挖个洞也够它的使用成本了。另外，用来瞎折腾也挺好的，变砖风险极低，进下 Maskrom 模式即可救回。

不过，这里需要注意两个坑

1. 最好是在 win 实体机上烧录固件
2. 换条好点的 usb 公对公数据线

上次遇到因为数据线问题造成固件无法刷入，还是使用 checkra1n 对 iphone 手机进行越狱，那个情况是用 typec 转 lightning 的数据线无法正常 patch，换成 usb 转 ligthning 的数据线就正常了，原因没有深究。 

（流下了没有技术的泪水.jpg

<br>
