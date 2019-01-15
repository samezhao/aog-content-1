---
title: "如何使用 ethtool 优化 Linux 虚拟机网卡性能"
description: "如何使用 ethtool 优化 Linux 虚拟机网卡性能"
author: Waynenus
resourceTags: 'Virtual Machines, Linux, Ethtool, NIC'
ms.service: virtual-machines
wacn.topic: aog
ms.topic: article
ms.author: wei.sun
ms.date: 12/31/2018
wacn.date: 12/31/2018
---

# 如何使用 ethtool 优化 Linux 虚拟机网卡性能

在高并发应用场景下，Linux 虚拟机可能会出现丢包现象，可以通过调整网卡的 Buffer size 来缓解此问题。

本文使用 ethtool 来查看和修改 RX/TX 值来获取更好性能，以下为参考示例：

在 ethtool 配置文件中可以看到，RX/TX 值的单位是 section size：

```shell
#define NETVSC_SEND_SECTION_SIZE                6144
#define NETVSC_RECV_SECTION_SIZE                1728
```

1. 查看当前网卡 RX/TX 参数：

    ```shell
    [root@centos75 ~]# ethtool -g eth0
    Ring parameters for eth0:
    Pre-set maximums:
    RX:                   18811
    RX Mini:              0
    RX Jumbo:             0
    TX:                   2560
    Current hardware settings:
    RX:                   10486
    RX Mini:              0
    RX Jumbo:             0
    TX:                   192
    ```

    > [!IMPORTANT]
    > Pre-set maximums 中的 RX/TX 值为该网卡的 Buffer size 最大值；
    > Current hardware settings 中 RX/TX 值代表该网卡当前的 Buffer size 大小。
    > 所以，设置的 Current hardware settings 的 RX/TX 值必须在 Pre-set maximums 的限制之内。

2. 调整 RX/TX 参数：

    ```shell
    [root@centos75 ~]# ethtool -G eth0 rx 18000 tx 2500
    ```

3. 查看调整后的 RX/TX 参数：

    ```shell
    [root@centos75 ~]# ethtool -g eth0
    Ring parameters for eth0:
    Pre-set maximums:
    RX:                   18811
    RX Mini:              0
    RX Jumbo:             0
    TX:                   2560
    Current hardware settings:
    RX:                   18000
    RX Mini:              0
    RX Jumbo:             0
    TX:                   2500
    ```

由于重启之后修改会失效，您可以在 *rc.local* 中设置为开机自动运行修改命令，参考如下：

1. 添加可执行权限：

    ```shell
    [root@centos75 ~]# chmod -x /etc/rc.d/rc.local
    ```

2. 添加修改命令，并保存：

    ```shell
    [root@centos75 ~]# vi /etc/rc.d/rc.local

    #!/bin/bash
    # THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
    #
    # It is highly advisable to create own systemd services or udev rules
    # to run scripts during boot instead of using this file.
    #
    # In contrast to previous versions due to parallel execution during boot
    # this script will NOT be run after all other services.
    #
    # Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
    # that this script will be executed during boot.

    touch /var/lock/subsys/local
    ethtool -G eth0 rx 18000 tx 2500
    ```

3. 重启并验证：

    ```shell
    [root@centos75 ~]# ethtool -g eth0
    Ring parameters for eth0:
    Pre-set maximums:
    RX:                   18811
    RX Mini:              0
    RX Jumbo:             0
    TX:                   2560
    Current hardware settings:
    RX:                   18000
    RX Mini:              0
    RX Jumbo:             0
    TX:                   2500
    ```