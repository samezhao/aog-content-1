---
title: 使用 PowerShell 脚本为 Windows 虚拟机开启 Azure 磁盘加密功能
description: 使用 PowerShell 脚本为 Windows 虚拟机开启 Azure 磁盘加密功能
services: ''
documentationCenter: ''
author: sscchh2001
manager: ''
editor: ''
tags: ''

ms.service: multiple
wacn.topic: aog
ms.topic: article
ms.author: chesh
ms.date: 12/13/2017
wacn.date: 06/21/2017
---

# 使用 PowerShell 脚本为 Windows 虚拟机开启 Azure 磁盘加密功能

本脚本提供了使用 PowerShell 脚本新建 Azure Key Vault 和 Azure AD 应用，并为现有 Windows 虚拟机开启 Azure 磁盘加密功能。

> [!TIP]
> Azure 备份服务现在已经支持为只使用加密密钥 ( BEK ) 加密的虚拟机执行备份和还原，使用密钥加密密钥 ( KEK ) 加密的虚拟机仍然可以被备份和还原。 

> [!TIP]
> Azure 虚拟机磁盘加密现在已经支持为使用托管磁盘的虚拟机进行加密。

> [!NOTE]
> 目前暂时不支持更新已加密的高级存储 VM 的加密设置。

创建 Azure 密钥保管库 :

```PowerShell
$RGName = "< yourResourcegroup >"
$Location = "<China East or China North>"
$VaultName= "<yourKeyVaultname>"
$KeyVault = New-AzureRmKeyVault -VaultName $VaultName -ResourceGroupName $RGName -Location $Location
```

设置 Azure AD 应用程序 :

为在 Azure 中正在运行的 VM 上启用加密时，Azure 磁盘加密将生成加密密钥并将其写入 Key Vault。 在 Key Vault 中管理加密密钥需要 Azure AD 身份验证，可以使用基于客户端机密的身份验证或基于客户端证书的 Azure AD 身份验证。请根据实际需要选择下列四种方式之一配置 Azure AD 应用程序。

1. 创建基于客户端机密的 Azure AD 应用程序 :
    
    ```PowerShell
    $aadClientSecret = "<yourAadClientSecret>"
    $azureAdApplication = New-AzureRmADApplication -DisplayName "<Your Application Display Name>" -HomePage "<https://YourApplicationHomePage>" -IdentifierUris "<https://YouApplicationUri>" -Password $aadClientSecret
    $servicePrincipal = New-AzureRmADServicePrincipal –ApplicationId $azureAdApplication.ApplicationId
    ```

2. 为现有的 Azure AD 应用程序配置客户端机密

    若要执行以下命令，请获取并使用 [Azure AD PowerShell 模块](https://technet.microsoft.com/zh-cn/library/jj151815.aspx)。
    
    ```PowerShell
    $aadclientSecret = "<yourAadClientSecret>"
    $aadClientID = "<Client ID of your Azure AD application>"
    Connect-MsolService -AzureEnvironment azurechinacloud
    New-MsolServicePrincipalCredential -AppPrincipalId $aadClientID -Type password -Value $aadclientSecret
    ```

3. 创建基于证书的 Azure AD 应用程序 :

    > [!TIP]
    > 请将 `<yourpassword>` 字符串替换为 pfx 证书的安全密码。

    ```PowerShell
    $Cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate("C:\certificates\examplecert.pfx", "<yourpassword>")
    $CertValue = [System.Convert]::ToBase64String($Cert.GetRawCertData())
    $azureAdApplication = New-AzureRmADApplication -DisplayName "<Your Application Display Name>" -HomePage "<https://YourApplicationHomePage>" -IdentifierUris "<https://YouApplicationUri>" -CertValue $CertValue -StartDate $Cert.StartDate -EndDate $Cert.EndDate
    $AADClientID = $AzureAdApplication.ApplicationId
    $aadClientCertThumbprint= $cert.Thumbprint
    $servicePrincipal = New-AzureRmADServicePrincipal –ApplicationId $AADClientID
    ```

4. 为现有的 Azure AD 应用程序配置证书 :

    若要执行以下命令，请获取并使用 [Azure AD PowerShell 模块](https://technet.microsoft.com/zh-cn/library/jj151815.aspx)。

    > [!TIP]
    > 请将 `<yourpassword>` 字符串替换为 pfx 证书的安全密码。

    ```PowerShell
    $Cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate("C:\certificates\examplecert.pfx", "<yourpassword>")
    $CertValue = [System.Convert]::ToBase64String($Cert.GetRawCertData())
    $aadClientID = "<Client ID of your Azure AD application>"
    $aadClientCertThumbprint= $cert.Thumbprint
    Connect-MsolService -AzureEnvironment azurechinacloud
    New-MsolServicePrincipalCredential -AppPrincipalId $aadClientID -Type asymmetric -Value $credValue -Usage verify
    ```

将 PFX 证书文件上传到 Key Vault, 机密名变量 `$keyVaultSecretName` 的值可以任意设置。 

> [!NOTE]
> 使用基于客户端机密的身份验证可跳过这一步。  

> [!TIP]
> 请将 `<yourpassword>` 字符串替换为 pfx 证书的安全密码。

```PowerShell
$certLocalPath = "C:\certificates\examplecert.pfx"
$certPassword = "<yourpassword>"
$resourceGroupName = "<yourResourcegroup>"
$keyVaultName = "<yourKeyVaultName>"
$keyVaultSecretName = "<yourAadCertSecretName>"

$fileContentBytes = get-content $certLocalPath -Encoding Byte
$fileContentEncoded = [System.Convert]::ToBase64String($fileContentBytes)
$jsonObject = @"
{
"data": "$filecontentencoded",
"dataType": "pfx",
"password": "$certPassword"
}
"@
$jsonObjectBytes = [System.Text.Encoding]::UTF8.GetBytes($jsonObject)
$jsonEncoded = [System.Convert]::ToBase64String($jsonObjectBytes)
$secret = ConvertTo-SecureString -String $jsonEncoded -AsPlainText -Force
Set-AzureKeyVaultSecret -VaultName $keyVaultName -Name $keyVaultSecretName -SecretValue $secret
Set-AzureRmKeyVaultAccessPolicy -VaultName $keyVaultName -ResourceGroupName $resourceGroupName –EnabledForDeployment
```

将 Key Vault 中的证书部署到现有 VM。

> [!NOTE]
> 使用基于客户端机密的身份验证可跳过这一步。

```PowerShell
$resourceGroupName = "<yourResourcegroup>"
$keyVaultName = "<yourKeyVaultName>"
$keyVaultSecretName = "<yourAadCertSecretName>"
$vmName = "<yourVMName>"
$certUrl = (Get-AzureKeyVaultSecret -VaultName $keyVaultName -Name $keyVaultSecretName).Id
$sourceVaultId = (Get-AzureRmKeyVault -VaultName $keyVaultName -ResourceGroupName $resourceGroupName).ResourceId
$vm = Get-AzureRmVM -ResourceGroupName $resourceGroupName -Name $vmName
$vm = Add-AzureRmVMSecret -VM $vm -SourceVaultId $sourceVaultId -CertificateStore "My" -CertificateUrl $certUrl
Update-AzureRmVM -VM $vm -ResourceGroupName $resourceGroupName
```

为 Azure AD 应用程序设置 Key Vault 访问策略 :

```PowerShell
$keyVaultName = "<yourKeyVaultName>"
$AADClientID = $AzureAdApplication.ApplicationId 
$RGName = "<yourResourceGroup>"
Set-AzureRmKeyVaultAccessPolicy -VaultName $keyVaultName -ServicePrincipalName $AADClientID -PermissionsToKeys "WrapKey" -PermissionsToSecrets "Set" -ResourceGroupName $RGName
```

设置密钥加密密钥（可选）:

```PowerShell
$KEKName = "<yourKEKName>"
$KEK = Add-AzureKeyVaultKey -VaultName $VaultName -Name $KEKName -Destination "Software"
$KeyEncryptionKeyUrl = $KEK.Key.kid
```

设置 Key Vault 权限 :

```PowerShell
$keyVaultName = "<yourKeyVaultName>"
$RGName = "<yourResourceGroup>"
Set-AzureRmKeyVaultAccessPolicy -VaultName $keyVaultName -ResourceGroupName $RGName -EnabledForDiskEncryption
```

为 Azure VM 开启磁盘加密功能

请根据之前配置的 Azure AD 应用程序和密钥加密密钥的情况选择以下四种方式之一配置 Azure VM 磁盘加密功能。

1. 使用基于客户端机密的身份验证

    ```PowerShell
    Set-AzureRmVMDiskEncryptionExtension -ResourceGroupName $rgname -VMName $vmName -AadClientID $aadClientID -AadClientSecret $aadClientSecret -DiskEncryptionKeyVaultUrl $diskEncryptionKeyVaultUrl -DiskEncryptionKeyVaultId $KeyVaultResourceId
    ```

2. 使用基于客户端机密的身份验证，并使用密钥加密密钥

    ```PowerShell
    Set-AzureRmVMDiskEncryptionExtension -ResourceGroupName $RGName -VMName $vmName -AadClientID $AADClientID -AadClientSecret $AADClientSecret -DiskEncryptionKeyVaultUrl $DiskEncryptionKeyVaultUrl -DiskEncryptionKeyVaultId $KeyVaultResourceId -KeyEncryptionKeyUrl $KeyEncryptionKeyUrl -KeyEncryptionKeyVaultId $KeyVaultResourceId
    ```

3. 使用基于证书的身份验证

    ```PowerShell
    Set-AzureRmVMDiskEncryptionExtension -ResourceGroupName $RGName -VMName $VMName -AadClientID $AADClientID -AadClientCertThumbprint $AADClientCertThumbprint -DiskEncryptionKeyVaultUrl $DiskEncryptionKeyVaultUrl -DiskEncryptionKeyVaultId $KeyVaultResourceId
    ```

4. 使用基于证书的身份验证，并使用密钥加密密钥

    ```PowerShell
    Set-AzureRmVMDiskEncryptionExtension -ResourceGroupName $RGName -VMName $VMName -AadClientID $AADClientID -AadClientCertThumbprint $AADClientCertThumbprint -DiskEncryptionKeyVaultUrl $DiskEncryptionKeyVaultUrl -DiskEncryptionKeyVaultId $KeyVaultResourceId -KeyEncryptionKeyUrl $KeyEncryptionKeyUrl -KeyEncryptionKeyVaultId $KeyVaultResourceId
    ```

查看当前订阅内的VM 磁盘加密状态

```PowerShell
$osVolEncrypted = {(Get-AzureRmVMDiskEncryptionStatus -ResourceGroupName $_.ResourceGroupName -VMName $_.Name).OsVolumeEncrypted}
$dataVolEncrypted= {(Get-AzureRmVMDiskEncryptionStatus -ResourceGroupName $_.ResourceGroupName -VMName $_.Name).DataVolumesEncrypted}
Get-AzureRmVm | Format-Table @{Label="MachineName"; Expression={$_.Name}}, @{Label="OsVolumeEncrypted"; Expression=$osVolEncrypted}, @{Label="DataVolumesEncrypted"; Expression=$dataVolEncrypted}
```

## 脚本链接

[Enforce-DiskEncryptionwithKEK](https://github.com/wacn/AOG-CodeSample/tree/master/PowerShell/Enforce-DiskEncryptionwithKEK.ps1)
