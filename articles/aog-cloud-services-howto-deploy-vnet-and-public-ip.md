---
title: "云服务配置虚拟网络和公共 IP"
description: "云服务配置虚拟网络和公共 IP"
author: chenzheng1988
resourceTags: 'Cloud Services, Virtual Network, Public IP'
ms.service: cloud-services
wacn.topic: aog
ms.topic: article
ms.author: v-ciwu
ms.date: 12/25/2018
wacn.date: 12/25/2018
---

# 云服务配置虚拟网络和公共 IP

用户将应用迁移至 Azure 云服务时，很重要的一点考率是保证服务的安全性，在 Azure 上，保护应用程序资源的最佳实践就是创建虚拟机网络和子网。

用户在创建云服务时会生成一个 Virtual IP Address (VIP)，而 VIP 是被云服务后端所有的机器共享， 如果用户想要能够通过特定的IP访问云服务中的某个实例，可以为经典云服务的实例请求 ILPIP。

本文介绍了如何为经典云服务配置虚拟网络、配置 ILPIP 以及在配置过程中的注意事项:

## 经典云服务配置虚拟网络

经典云服务只可以配置经典虚拟网络，经典云服务需要在云服务的配置文件 .cscfg 中添加 NetworkConfiguration，并且需要重新部署做 full deployment 才能生效。需要注意检查添加相关配置之后的 xml 文件是否为有效文件，可以通过在线工具 [XML Beautifier](http://xmlbeautifier.com/) 生成有效 xml 文件。

参考示例如下：

```xml
<ServiceConfiguration>  
  <NetworkConfiguration>  
    <VirtualNetworkSite name="经典虚拟网络的虚拟网络站点名称"/>  
    <AddressAssignments>  
      <InstanceAddress roleName="角色名称">  
        <Subnets>  
          <Subnet name="子网名称"/>  
        </Subnets>  
      </InstanceAddress>
    </AddressAssignments>  
  </NetworkConfiguration>  
</ServiceConfiguration>
```

> [!NOTE]
> 经典虚拟网络的站点名称需要您在 [Azure 门户](https://portal.azure.cn) 中找到已创建的经典虚拟网络，然后选择概述，其中**( .cscfg 文件的)虚拟网络站点名称**中的值就是经典虚拟网络的虚拟网络站点名称，如下图所示:
> ![01](media/aog-cloud-services-howto-deploy-vnet-and-public-ip/01.png "01")

云服务配置虚拟网络请参考：[NetworkConfiguration Schema](https://docs.microsoft.com/zh-cn/previous-versions/azure/reference/jj156091%28v%3dazure.100%29)。

创建经典虚拟机网络请参考：[使用 Azure 门户创建虚拟网络（经典）](https://docs.azure.cn/zh-cn/virtual-network/virtual-networks-create-vnet-classic-pportal)。

## 经典云服务配置公共 IP

可以通过以下方式配置 ILPIP，上传到云服务后系统会自动创建 ILPIP，只能为每个 经典云服务角色实例分配一个 ILPIP，每个订阅最多可使用 5 个 ILPIP。

1. 下载云服务的 .cscfg 文件。

2. 修改云服务的配置文件 .cscfg，添加 InstanceAddress 节点，并且为 webrole 配置 ILPIP。

    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <ServiceConfiguration serviceName="ILPIPSample" xmlns="http://schemas.microsoft.com/ServiceHosting/2008/10/ServiceConfiguration" osFamily="4" osVersion="*" schemaVersion="2014-01.2.3">
      <Role name="WebRole1">
        <Instances count="1" />
            <ConfigurationSettings>
        <Setting name="Microsoft.WindowsAzure.Plugins.Diagnostics.ConnectionString" value="UseDevelopmentStorage=true" />
            </ConfigurationSettings>
      </Role>
      <NetworkConfiguration>
       <AddressAssignments>
          <InstanceAddress roleName="WebRole1">
            <PublicIPs>
              <PublicIP name="MyPublicIP" domainNameLabel="MyPublicIP" />
            </PublicIPs>
          </InstanceAddress>
        </AddressAssignments>
      </NetworkConfiguration>
    </ServiceConfiguration>
    ```

3. 上传该 .cscfg 文件到云服务

    详细介绍请参考：[实例层级公共 IP (经典)概述](https://docs.azure.cn/zh-cn/virtual-network/virtual-networks-instance-level-public-ip#manage-an-ilpip-for-a-cloud-services-role-instance)。

> [!NOTE]
> 如果在配置了虚拟网络的情况下配置 ILPIP 需要按照顺序填写，先是 *Subnets* 然后 *PublicIPs*，参考如下：
> 
> ```xml
> <NetworkConfiguration>
>     <VirtualNetworkSite name="Group testgroup czclassicvnet" />
>     <AddressAssignments>
>         <InstanceAddress roleName="ContosoAdsWeb">
>             <Subnets>
>                 <Subnet name="default" />
>             </Subnets>
>             <PublicIPs>
>                 <PublicIP name="mypublicip" domainNameLabel="czpublicip" />
>             </PublicIPs>
>         </InstanceAddress>
>     </AddressAssignments>
> </NetworkConfiguration>
>```