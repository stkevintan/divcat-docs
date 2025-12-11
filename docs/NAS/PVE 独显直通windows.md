---
tags:
  - FnOS
  - NAS
  - PVE
date: 2025-12-11
---
## Grub配置

参考 [[PVE 集成显卡直通飞牛OS#更新Grub]]

## 安装windows并启用远程登陆

首先我们使用默认显卡来安装windows （这样我们可以通过PVE console来查看当前画面）
安装过程可以参考 [1]

## 开机加载`vfio`module

编辑`/etc/modules`，加入以下內容

```bash
vfio
vfio_iommu_type1
vfio_pci
```

也许在其他教程中还会多加一个`vfio_virqfd`，不过在 kernel6.2 之后这个模块不再需要添加了，已经包含在`vfio`中了

编辑完成后使用`update-initramfs -u -k all`来更新 initramfs

## 配置vfio

查找需要直通的显卡、声卡id：

```bash
lspci -D -nn | grep VGA
lspci -D -nn | grep Audio
0000:01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD106M [GeForce RTX 4070 Max-Q / Mobile] [10de:2860] (rev a1)
0000:c7:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Phoenix3 [1002:1900] (rev ba)
0000:01:00.1 Audio device [0403]: NVIDIA Corporation AD106M High Definition Audio Controller [10de:22bd] (rev a1)
0000:c7:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Radeon High Definition Audio Controller [Rembrandt/Strix] [1002:1640]
0000:c7:00.6 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Family 17h/19h/1ah HD Audio Controller 
```

记下需要直通的设备的 vendor id 和 device id,对于 NVIDIA,因此就是显卡 `10de:2860` 和 `10de:22bd`

编辑 `/etc/modprobe.d/vfio.conf` 来设置 vfio 参数,ids 后面添加的就是需要直通的设备 id,后面 softdep 是让内核在加载这些模块之前优先加载 vfio,避免被驱动模块抢先,这里只有 NVIDIA 的驱动模块,如果使用 intel 或者 amd 的显卡需要改为对应的模块

```bash
options vfio-pci ids=10de:0fc6,10de:0e1b
softdep nouveau pre: vfio-pci
softdep nvidia pre: vfio-pci
softdep nvidiafb pre: vfio-pci
softdep nvidia_drm pre: vfio-pci
softdep drm pre: vfio-pci
```

重新启动就可以开始配置显卡直通了

## 配置显卡直通

显卡：NVIDIA Corporation AD106M [GeForce RTX 4070 Max-Q / Mobile]

勾选 ROM-Bar, PCI-Express, Primary GPU 
![[image.png]]

声卡： NVIDIA Corporation AD106M High Definition Audio Controller

>💡我的情况并不需要配置vbios即可工作，有需要可以参考来提取并配置vbios: [1]


## Optional 忽略MSRS错误

```bash
echo "options kvm ignore_msrs=1 report_ignored_msrs=0" > /etc/modprobe.d/kvm.conf
```

- `ignore_msrs=1` - 忽略虚拟机对不支持的 MSR (Model Specific Register) 的访问
- `report_ignored_msrs=0` - 不在内核日志中报告被忽略的 MSR 访问

**MSR 是什么：**

MSR (Model Specific Register) 是 CPU 特定的寄存器，不同的 CPU 型号有不同的 MSR。虚拟机可能会尝试访问宿主机 CPU 不支持的 MSR。

**为什么需要这些参数：**

默认情况下，当虚拟机访问不存在的 MSR 时：

- KVM 会拒绝访问并可能导致虚拟机崩溃
- 内核日志会被大量错误信息刷屏

**使用场景：**

常见于：

1. 运行 Windows 虚拟机（Windows 经常访问各种 MSR）
2. CPU 直通或嵌套虚拟化
3. 虚拟机 CPU 型号与宿主机不完全匹配

参考资料：
[1]:  [以铭凡N5-255为例，PVE9下AMD核显直通虚拟Win教程，不花屏不死机](https://diyforfun.cn/1552.html) 
[2]: [在 Proxmox 上進行 PCI-E 直通](https://www.zenwen.tw/pci-passthrough-with-proxmox)
[3]: [Proxmox VE 8.4 显卡直通完整指南：NVIDIA 2080 Ti 实战背景： PCIe Passthroug - 掘金](https://juejin.cn/post/7499089664566493210)