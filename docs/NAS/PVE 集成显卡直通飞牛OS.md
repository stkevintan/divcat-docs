---
tags:
  - NAS
  - PVE
  - FnOS
date: 2025-12-11
---

## 更新Grub

编辑 `/etc/default/grub`

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt initcall_blacklist=sysfb_init pcie_acs_override=downstream,multifunction"
```

> 💡 pve内核经常更新，amd_iommu=on有的内核已经内置，因此不需要上面加amd_iommu=on。如果遇到更新pve内核后，提示iommu没有开启，请手动在以上此处添加amd_iommu=on，并update-grub重启后即可解决。


1. `iommu=pt` : 仅对直通设备启用IOMMU，减少性能开销
2. `initcall_blacklist=sysfb_init` :启动时运行黑名单内添加项，在Intel的机型中，此项非必要添加，但是在amd机型中建议添加，否则会影响核显直通后的性能，比如4K60Hz降低到30Hz。 防止宿主机占用显卡帧缓冲区
3. `pcie_acs_override=downstream,multifunction` 解决某些PCIe设备的ACS限制问题，避免死机

编辑 `/etc/modprobe.d/pve-blacklist.conf`

```bash
blacklist nvidiafb
blacklist amdgpu
options vfio_iommu_type1 allow_unsafe_interrupts=1
```

解释：屏蔽amd显卡驱动， `options vfio_iommu_type1` `allow_unsafe_interrupts=1` 允许不安全的设备中断，在部分机型上不加此项会导致虚拟机启动加载转圈时直接宿主PVE卡死。


> 注意：在设备黑名单中添加amdgpu，并grub里开启引导自动加载后，pve启动时会卡在以下界面。是正常的！等待一会只要PVE可以通过网页访问到WebUI管理页面就是正常没问题的。因为PVE开机时会按照设备黑名单设置的，释放对核显的控制权。这样才能在核显直通给虚拟Win的时候避免冲突而导致的无法加载驱动，错误代码43
![[image 1.png]]



更新grub和initramfs并重启

```bash
update-grub
update-initramfs -u -k
reboot
```

## 创建VM 安装飞牛

我们先用虚拟显卡把飞牛装好：

- VM配置
    - 显卡：Default
    - 机型：q35
    - BIOS：OVMF
    - EFI分区：UEFI（OVMF）需要
    - TPM设备：No
    - 磁盘：SCSI 大小100G（按需设置，或硬盘直通）
    - CPU：host 核心数量8（按需设置）
    - 内存：4G及以上（核显直通建议）
    - 网络：virtIO（半虚拟化或网卡直通）网卡

## 显卡直通

当系统安装成功，我们可以通过<ip>访问飞牛之后，进行直通：

1. 显卡改成 none (这一步之后，就无法在pve的console中看到虚拟机画面了）
2. **添加PCI设备：**
    - 添加显卡（pciid不固定）pcie设备里面勾选：**主gpu，rom-bar，pcie-express这三个选项，所有功能：不勾选**
    - 添加声卡（pciid不固定）

可以通过下列命令来确定显卡和声卡设备：

```shell
lspci -D -nn | grep VGA
lspci -D -nn | grep Audio
....
Advanced Micro Devices, Inc. [AMD/ATI] Phoenix3
Advanced Micro Devices, Inc. [AMD/ATI] Radeon High Definition Audio Controller [Rembrandt/Strix]
```


> 💡注意：显卡的PCI组都会对应绑定着一个声卡，需要两者同时直通才能成功。

## 参考资料
1. <https://diyforfun.cn/1552.html>