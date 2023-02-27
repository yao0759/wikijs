---
title: slurm--容器指南
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出
published: true
date: 2023-02-27T03:51:38.469Z
tags: slurm, container, docker
editor: markdown
dateCreated: 2023-01-02T14:47:16.399Z
---


容器正在HPC工作负载中被采用。容器依靠现有的内核功能，允许用户对应用程序在任何时候看到的和可以交互的内容进行更大的控制。对于HPC工作负载，这些通常被限制在挂载命名空间。Slurm原生支持为作业和步骤请求无特权的OCI容器。

## 已知的限制
以下是Slurm OCI容器实现的已知限制列表。

- 所有容器必须在无特权（即无根）的调用下运行。所有的命令都是由Slurm作为用户调用的，没有特殊权限。

- 不支持自定义容器网络。所有的容器都应该在 "主机 "网络中工作。
- Slurm不会将OCI容器包转移到执行节点上。捆绑包必须已经存在于执行节点的请求路径上。
- 容器受到所使用的OCI运行时的限制。如果运行时不支持某项功能，那么该功能将不能用于任何使用容器的工作。
- oci.conf必须在作业的执行节点上进行配置，否则所请求的容器将被Slurm忽略（但可以被作业或任何给定的插件使用）。

## 前提条件

主机内核必须被配置为允许用户登陆容器。

```
$ sudo sysctl -w kernel.unprivileged_userns_clone=1
```

Docker还提供了一个工具来验证内核的配置。

```
$ dockerd-rootless-setuptool.sh check --force
[INFO] Requirements are satisfied
```

## 所需的软件。

- 功能齐全的OCI运行时。它首先需要能够在Slurm之外运行。
- 功能齐全的OCI捆绑生成工具。Slurm需要符合OCI容器的捆绑作业。

## 各种OCI运行时的配置实例

OCI运行时规范为所有符合要求的运行时提供了要求，但并没有明确规定运行时如何使用参数的要求。为了支持尽可能多的运行时，Slurm为每个OCI运行时操作发出的命令提供模式替换。这将允许一个网站根据需要编辑OCI运行时的调用方式，以确保兼容性。

对于runc和crun，提供了两套例子。OCI运行时规范只提供了启动和创建操作的顺序，但这些运行时提供了一个更有效的运行操作。强烈建议网站使用运行操作（如果提供的话），因为启动和创建操作需要Slurm轮询OCI运行时，以了解容器何时完成执行。虽然Slurm试图尽可能地提高轮询的效率，但这将导致一个线程在作业中使用CPU时间，并且Slurm在捕捉容器执行完成时的反应较慢。

所提供的例子经过测试是可行的，但只是建议。网站应确保所使用的根目录是安全的，不会被跨用户查看和修改。所提供的例子指向"/run/user/%U"，其中%U将被替换为数字用户ID，它应该由systemd独立于Slurm创建和管理。

- runc使用create/start。

  ```
  RunTimeQuery="runc --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
  RunTimeCreate="runc --rootless=true --root=/run/user/%U/ create %n.%u.%j.%s.%t -b %b"
  RunTimeStart="runc --rootless=true --root=/run/user/%U/ start %n.%u.%j.%s.%t"
  RunTimeKill="runc --rootless=true --root=/run/user/%U/ kill -a %n.%u.%j.%s.%t"
  RunTimeDelete="runc --rootless=true --root=/run/user/%U/ delete --force %n.%u.%j.%s.%t"
  ```

- runc使用run（推荐）

  ```
  RunTimeQuery="runc --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
  RunTimeQuery="runc --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
  RunTimeKill="runc --rootless=true --root=/run/user/%U/ kill -a %n.%u.%j.%s.%t"
  RunTimeDelete="runc --rootless=true --root=/run/user/%U/ delete --force %n.%u.%j.%s.%t"
  RunTimeRun="runc --rootless=true --root=/run/user/%U/ run %n.%u.%j.%s.%t -b %b"
  ```

- crun使用create/start

  ```
  RunTimeQuery="crun --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
  RunTimeKill="crun --rootless=true --root=/run/user/%U/ kill -a %n.%u.%j.%s.%t"
  RunTimeDelete="crun --rootless=true --root=/run/user/%U/ delete --force %n.%u.%j.%s.%t"
  RunTimeCreate="crun --rootless=true --root=/run/user/%U/ create --bundle %b %n.%u.%j.%s.%t"
  RunTimeStart="crun --rootless=true --root=/run/user/%U/ start %n.%u.%j.%s.%t"
  ```

- crun使用run（推荐）

  ```
  RunTimeQuery="crun --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
  RunTimeKill="crun --rootless=true --root=/run/user/%U/ kill -a %n.%u.%j.%s.%t"
  RunTimeDelete="crun --rootless=true --root=/run/user/%U/ delete --force %n.%u.%j.%s.%t"
  RunTimeRun="crun --rootless=true --root=/run/user/%U/ run --bundle %b %n.%u.%j.%s.%t"
  ```

- nvidia-container-runtime使用create/start

  ```
  RunTimeQuery="nvidia-container-runtime --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
  RunTimeCreate="nvidia-container-runtime --rootless=true --root=/run/user/%U/ create %n.%u.%j.%s.%t -b %b"
  RunTimeStart="nvidia-container-runtime --rootless=true --root=/run/user/%U/ start %n.%u.%j.%s.%t"
  RunTimeKill="nvidia-container-runtime --rootless=true --root=/run/user/%U/ kill -a %n.%u.%j.%s.%t"
  RunTimeDelete="nvidia-container-runtime --rootless=true --root=/run/user/%U/ delete --force %n.%u.%j.%s.%t"
  ```

- nvidia-container-runtime使用run（推荐）

  ```
  RunTimeQuery="nvidia-container-runtime --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
  RunTimeQuery="nvidia-container-runtime --rootless=true --root=/run/user/%U/ state %n.%u.%j.%s.%t"
  RunTimeKill="nvidia-container-runtime --rootless=true --root=/run/user/%U/ kill -a %n.%u.%j.%s.%t"
  RunTimeDelete="nvidia-container-runtime --rootless=true --root=/run/user/%U/ delete --force %n.%u.%j.%s.%t"
  RunTimeRun="nvidia-container-runtime --rootless=true --root=/run/user/%U/ run %n.%u.%j.%s.%t -b %b"
  ```

- hpcng singularity v3.8.0:

  ```
  OCIRunTimeQuery="sudo singularity oci state %n.%u.%j.%s.%t"
  OCIRunTimeCreate="sudo singularity oci create --bundle %b %n.%u.%j.%s.%t"
  OCIRunTimeStart="sudo singularity oci start %n.%u.%j.%s.%t"
  OCIRunTimeKill="sudo singularity oci kill %n.%u.%j.%s.%t"
  OCIRunTimeDelete="sudo singularity oci delete %n.%u.%j.%s.%t
  ```

  **警告：**Singuarity（v3.8.0）需要sudo来支持OCI，这是一个安全风险，因为用户能够修改这些调用。这个例子只是为了测试而提供的。

## 在Slurm之外测试OCI运行时
Slurm在作业步骤中直接调用OCI运行时。如果它失败了，那么作业也会失败。

- 转到包含OCI容器包的目录。

  ```
  cd $ABS_PATH_TO_BUNDLE
  ```

- 执行OCI容器运行时间。

  ```
  $OCIRunTime $ARGS create test --bundle $PATH_TO_BUNDLE
  $OCIRunTime $ARGS start test
  $OCIRunTime $ARGS kill test
  $OCIRunTime $ARGS delete test
  ```

  如果这些命令成功了，那么OCI运行时间就被正确配置了，可以在Slurm中进行测试。

## 请求容器作业或步骤
salloc、srun和sbatch（在Slurm 21.08+）有'--container'参数，可以用来请求容器运行时执行。请求的作业容器将不会被调用的步骤继承，不包括批处理和交互式步骤。

- 容器内的批处理步骤。

  ```
  sbatch --container $ABS_PATH_TO_BUNDLE --wrap 'bash -c "cat /etc/*rel*"'
  ```

- 容器内的批处理作业，步骤0。

  ```
  sbatch --wrap 'srun bash -c "--container $ABS_PATH_TO_BUNDLE cat /etc/*rel*"'
  ```

- 容器内的交互式步骤。

  ```
  salloc --container $ABS_PATH_TO_BUNDLE bash -c "cat /etc/*rel*"
  ```

- 容器内的交互式工作步骤0。

  ```
  salloc srun --container $ABS_PATH_TO_BUNDLE bash -c "cat /etc/*rel*"
  ```

- 容器内的工作步骤为0。

  ```
  srun --container $ABS_PATH_TO_BUNDLE bash -c "cat /etc/*rel*"
  ```

- 容器内有第1步的工作。

  ```
  srun srun --container $ABS_PATH_TO_BUNDLE bash -c "cat /etc/*rel*"
  ```

## OCI容器捆绑

有多种方法来生成一个OCI容器包。下面的说明是我们认为最简单的方法。OCI标准提供了任何给定捆绑的要求：[文件系统捆绑](https://github.com/opencontainers/runtime-spec/blob/main/bundle.md)

下面是关于如何使用一些不同的容器解决方案来生成容器的说明。

- 使用一个现有的工具在/image/rootfs中创建一个文件系统镜像：

  - debootstrap

    ```
    sudo debootstrap stable /run/user/%U/image/rootfs http://deb.debian.org/debian/
    ```

  - yum

    ```
    sudo yum --config /etc/yum.conf --installroot=/image/rootfs/ --nogpgcheck --releasever=${CENTOS_RELEASE} -y
    ```

  - docker

    ```shell
    mkdir -p ~/oci_images/alpine/rootfs
    cd ~/oci_images/
    docker pull alpine
    docker create --name alpine alpine
    docker export alpine | tar -C ~/oci_images/alpine/rootfs -xf -
    docker rm alpine
    ```

- 配置一个供Runtime执行的Bundle。

  - 使用runc来生成config.json。

  - ```
    cd ~/oci_images/alpine
    runc spec --rootless
    ```

  - 测试运行图像。

    ```
    srun --container ~/oci_images/alpine/ uptime
    ```

- 使用[umoci](https://github.com/opencontainers/umoci)和skopeo来生成一个完整的图像。

  ```
  mkdir -p ~/oci_images/
  cd ~/oci_images/
  skopeo copy docker://alpine:latest oci:alpine:latest
  umoci unpack --rootless --image alpine ~/oci_images/alpine
  srun --container ~/oci_images/alpine uptime
  ```

- 使用[singularity](https://docs.sylabs.io/guides/3.1/user-guide/oci_runtime.html)来生成一个完整的图像。

  ```
  mkdir -p ~/oci_images/alpine/
  cd ~/oci_images/alpine/
  singularity pull alpine
  sudo singularity oci mount ~/oci_images/alpine/alpine_latest.sif ~/oci_images/alpine
  mv config.json singularity_config.json
  runc spec --rootless
  srun --container ~/oci_images/alpine/ uptime
  ```

## 容器能够通过插件支持

Slurm还允许容器开发者创建SPANK插件，这些插件可以在工作执行的各个环节被调用，以支持容器。Slurm通常对基于SPANK的容器是不可知的，可以使其启动大多数（如果不是所有）类型的容器。任何使用插件来启动容器的网站都不应该创建或配置 "oci.conf "配置文件来停用OCI容器功能。
一些容器开发者只选择了一个命令行界面，这需要用户明确执行容器解决方案。

下面提供了几个第三方容器解决方案的链接。

- Charliecloud
- Docker
- UDOCKER
- Rootless Docker
- Kubernetes Pods (k8s)
- Shifter
- Singularity
- ENROOT
- Podman
- Sarus

请注意这个列表并不详尽，因为新的容器类型一直在创建。

## 容器类型

**Charliecloud**

Charliecloud是由LANL赞助的用户命名空间容器系统，提供HPC容器。Charliecloud支持以下内容。

- 由用户通过用户命名空间支持直接调用。

- 目前正在开发直接的Slurm支持。
- 有限的OCI图像支持（通过包装器）

**Docker（以root身份运行）**

Docker目前有多个设计点，使其对HPC系统不友好。通常阻止大多数网站使用Docker的问题是 "只允许受信任的用户控制你的Docker守护进程"[Docker Security]的要求，这对大多数HPC系统来说是不可接受的。

拥有可信用户的站点可以将他们加入docker Unix组，并允许他们直接从作业内部控制Docker。目前在Slurm中没有直接支持启动或停止docker容器。

建议网站从docker提取容器镜像（上面的程序），然后用Slurm运行容器。

**UDOCKER**

UDOCKER是Docker功能子集的克隆，旨在允许执行docker命令而不增加用户权限。

**Rootless Docker**

Rootless Docker（>=v20.10）不需要用户的额外权限，目前（截至2021年1月）没有用户获得权限的已知安全问题。每个用户需要在作业的每个节点上运行一个dockerd服务器的实例，以便使用docker。目前，Slurm没有辅助脚本或插件来自动建立或拆除docker守护程序。

建议网站从docker提取容器镜像（上面的程序），然后用Slurm运行容器。

**Kubernetes Pods (k8s)**

Kubernetes是一个使用Pods的容器编排系统，Pods通常是用于单一目的的容器的逻辑分组。

目前在Slurm中没有对Kubernetes Pods的直接支持。我们鼓励网站从Kubernetes提取OCI镜像，然后使用Slurm运行容器。用户可以使用sbatch中的"--dependency="参数创建一起启动的作业，以反映Pod的功能。用户还可以使用更大的分配，然后使用srun将每个pod作为一个平行步骤启动。

**Shifter**

Shifter是NERSC的一个容器项目，为HPC容器提供完全的调度器集成。

- Shifter提供了与Slurm集成的[完整说明](https://slurm.schedmd.com/SLUG17/SLUG_Bull_Singularity.pdf)。

- 关于Shifter和Slurm的介绍。

  - [Never Port Your  Code Again – Docker functionality with  Shifter using  SLURM](https://slurm.schedmd.com/SLUG15/shifter.pdf)

  - [Shifter: Containers in HPC Environments (slideshare.net)](https://www.slideshare.net/insideHPC/shifter-containers-in-hpc-environments)

**Singularity**

Singularity是混合容器系统，它支持。

- Slurm通过插件集成（针对singularity v2.x）。在SLUG17的Singularity演讲中提供了对该插件的完整描述。
- 通过沙盒模式的用户命名空间容器，不需要额外的权限。
- 用户通过Slurm外的setuid可执行文件直接调用singularity。

**ENROOT**

[Enroot](https://github.com/NVIDIA/enroot)是一个由NVIDIA赞助的用户名称空间容器系统，它支持：

- 通过[pyxis](https://github.com/NVIDIA/pyxis)整合Slurm
- 对Nvidia GPU的本地支持
- 更快的Docker镜像导入

**Podman**

Podman是一个由Redhat/IBM赞助的用户命名空间容器系统，它支持：

- 替换Docker。
- 由用户直接调用。(目前缺乏对Slurm的直接支持）。
- 通过[buildah](https://buildah.io/)建立无根的镜像
- 本地OCI镜像支持

**Sarus**

[Sarus](https://github.com/eth-cscs/sarus)是一个由苏黎世联邦理工学院CSCS赞助的特权容器系统，它支持：

- [通过OCI钩子进行Slurm镜像同步](https://sarus.readthedocs.io/en/latest/config/slurm-global-sync-hook.html)
- 本地OCI图像支持
- 支持NVIDIA GPU
- 与Shifter类似的设计