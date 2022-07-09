---
title: Linux Enterprise Server 15 SP3配置安装kvm nvidia vGPU
description: 
published: true
date: 2022-07-09T08:11:38.886Z
tags: install, vgpu, 显卡, suse, kvm
editor: markdown
dateCreated: 2022-06-23T07:00:46.996Z
---

**简介：** Linux Enterprise Server 15 SP3配置安装kvm nvidia vGPU

## 参考链接

-   [NVIDIA virtual GPU for KVM guests | SUSE Linux Enterprise Server 15 SP3](https://documentation.suse.com/sles/15-SP3/html/SLES-all/article-nvidia-vgpu.html)  
    
-   [deployment-guide-vgpu-Ampere-GPU.pdf](http://vgpu.com.cn/DeploymentGuide/deployment-guide-vgpu-Ampere-GPU.pdf)  
    

## 配置过程

### 配置要求

-   BIOS启用SRIOV  
    
-   BIOS启用Above 4G encoding  
    
-   BIOS启用Intel VT-d  
    

更详细信息可以参考一下链接

-   [NVIDIA® Virtual GPU Software Supported GPUs](https://docs.nvidia.com/grid/gpus-supported-by-vgpu.html)  
    
-   [NVIDIA Virtual GPU (vGPU) Software Documentation](https://docs.nvidia.com/grid/index.html)  
    

然后启用IOMMU，添加到grub上

```
cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-default [...] intel_iommu=on [...]
```

假如没有上述iommu字段，那就要添加/etc/default/grub

-   Intel cpu

```
GRUB_CMDLINE_LINUX="intel_iommu=on"
```

-   amd cpu

```
GRUB_CMDLINE_LINUX="amd_iommu=on"
```

生成一个新的grub文件

```
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

然后禁用 nouveau kernel module

```
echo "blacklist nouveau" > /etc/modprobe.d/50-blacklist.conf
```

重启

### 安装

在安装依赖包前，建议先安装国内源，且把旧的源删除

清理缓存

安装 kernel-default-devel

```
zypper update
zypper install kernel-default-devel dkms
zypper install -t pattern devel_C_C++ devel_kernely
```

切换init模式

安装vGPU驱动，这个驱动跟常规的驱动不一样，跟厂商那边获取

```
chmod +x NVIDIA-Linux-x86_64-470.82-vgpu-kvm.run
./NVIDIA-Linux-x86_64-470.82-vgpu-kvm.run --dkms
```

查看是否安装成功，假如能正确显示显卡信息，则代表安装成功

```
localhost:~ 
Tue Dec  7 16:14:42 2021
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.82       Driver Version: 470.82       CUDA Version: N/A      |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA RTX A5000    On   | 00000000:18:00.0 Off |                  Off |
| 52%   78C    P0   203W / 230W |  23936MiB / 24258MiB |     89%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   1  NVIDIA RTX A5000    On   | 00000000:3B:00.0 Off |                  Off |
| 54%   78C    P0   199W / 230W |  23936MiB / 24258MiB |     74%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A     14174    C+G   vgpu                            23936MiB |
|    1   N/A  N/A     14174    C+G   vgpu                            23936MiB |
+-----------------------------------------------------------------------------+
```

还需要查看nvidia module

```
localhost:~ 
nvidia_vgpu_vfio       69632  36
nvidia              35364864  3722
mdev                   28672  2 vfio_mdev,nvidia_vgpu_vfio
vfio                   40960  8 vfio_mdev,nvidia_vgpu_vfio,vfio_iommu_type1
drm                   614400  7 drm_kms_helper,drm_vram_helper,ast,nvidia,drm_ttm_helper,ttm
```

nvidia vGPU有两种模式：

-   profile  
    
-   mig  
    

首先A5000不支持MIG模式，以下输出信息可知

```
localhost:~ 
Unable to enable MIG Mode for GPU 00000000:18:00.0: Not Supported
Treating as warning and moving on.
All done.
```

所以我们需要使用的profile的配置方式

在刚安装完驱动后，先重启机器，重启后，启用显卡设备

```
sudo /usr/lib/nvidia/sriov-manage -e 00:18:0000.0
sudo /usr/lib/nvidia/sriov-manage -e 00:3b:0000.0
cd /sys/bus/pci/devices/0000:18:00.0/virtfn0/mdev_supported_types
for i in *; do echo "" $(cat $i/name) available: $(cat $i/avail*); done
NVIDIA RTXA5000-1B available: 0
 NVIDIA RTXA5000-2B available: 0
 NVIDIA RTXA5000-1Q available: 0
 NVIDIA RTXA5000-2Q available: 0
 NVIDIA RTXA5000-3Q available: 0
 NVIDIA RTXA5000-4Q available: 0
 NVIDIA RTXA5000-6Q available: 0
 NVIDIA RTXA5000-8Q available: 0
 NVIDIA RTXA5000-12Q available: 0
 NVIDIA RTXA5000-24Q available: 0
 NVIDIA RTXA5000-1A available: 0
 NVIDIA RTXA5000-2A available: 0
 NVIDIA RTXA5000-3A available: 0
 NVIDIA RTXA5000-4A available: 0
 NVIDIA RTXA5000-6A available: 0
 NVIDIA RTXA5000-8A available: 0
 NVIDIA RTXA5000-12A available: 0
 NVIDIA RTXA5000-24A available: 0
 NVIDIA RTXA5000-4C available: 0
 NVIDIA RTXA5000-6C available: 0
 NVIDIA RTXA5000-8C available: 0
 NVIDIA RTXA5000-12C available: 0
 NVIDIA RTXA5000-24C available: 0
uuidgen
f715f63c-0d00-4007-9c5a-b07b0c6c05de
sudo echo "f715f63c-0d00-4007-9c5a-b07b0c6c05de" > nvidia-666/create
sudo dmesg | tail
[...]
[ 3218.491843] vfio_mdev f715f63c-0d00-4007-9c5a-b07b0c6c05de: Adding to iommu group 322
[ 3218.499700] vfio_mdev f715f63c-0d00-4007-9c5a-b07b0c6c05de: MDEV: group_id = 322
[ 3599.608540] vfio_mdev f715f63c-0d00-4007-9c5a-b07b0c6c05de: Removing from iommu group 322
[ 3599.616753] vfio_mdev f715f63c-0d00-4007-9c5a-b07b0c6c05de: MDEV: detaching iommu
[ 3626.345530] vfio_mdev f715f63c-0d00-4007-9c5a-b07b0c6c05de: Adding to iommu group 322
[ 3626.353383] vfio_mdev f715f63c-0d00-4007-9c5a-b07b0c6c05de: MDEV: group_id = 322
```