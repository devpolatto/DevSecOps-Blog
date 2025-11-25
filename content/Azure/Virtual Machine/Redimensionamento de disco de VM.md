---
title: Redimensionamento de disco de VM
tags:
  - Azure
  - VirtualMachine
  - Disk
enableToc: true
---

Esta documentação fornece instruções passo a passo para redimensionar um disco gerenciado (Data Disk) em uma máquina virtual (VM) do Azure.

# Pré-requisitos
- Powershell ou Shell no linux
- **Módulo do Azure PowerShell**: Certifique-se de que tem o módulo do Azure PowerShell instalado. Caso contrário, instale-o usando o comando abaixo:
---

```powershell
Install-Module -Name Az -AllowClobber -Scope CurrentUser
```
# Passos para redimensionar um disco gerido

## Redimensionar o disco gerenciado.

1. Conecte-se à sua conta do Azure
	Abra o terminal do PowerShell e Conecte-s à sua conta do Azure utilizando o seguinte comando:

```Powershell
Connect-AzAccount
```


O termianl irá solicitarque  introduza as suas credenciais do Azure.

2. Selecione a Subscription apropriada
	Selecione a assinatura do Azure que contém a VM para a qual você deseja redimensionar o disco:
	
```
Select-AzSubscription –SubscriptionName 'my-subscription-name'
```

Substitua 'my-subscription-name' pelo nome da sua subscrição.

3. Definir grupo de recursos e nome da VM
	Defina variáveis para o nome do grupo de recursos e o nome da VM:
	
```
$rgName = 'prod-br-acquirer-rg'
$vmName = 'prod-br-acquirer-sftp-vm'
```

4. Recuperar o objeto VM
	Recupere o objeto VM usando o seguinte comando:

```
$vm = Get-AzVM -ResourceGroupName $rgName -Name $vmName
```

5. Parar a VM
	Antes de redimensionar o disco, é necessário parar a VM. Use o comando abaixo para parar a VM:

```
Stop-AzVM -ResourceGroupName $rgName -Name $vmName
```


> [!WARNING] Importante
> Aguarde até que a VM seja completamente parada antes de prosseguir.


6. Redimensionar o disco
	Defina o novo tamanho do disco em gigabytes (GB). Por exemplo, para redimensionar o disco para 200 GB:

```
$vm.StorageProfile.OSDisk.DiskSizeGB = 200
```

7. Atualizar a configuração da VM
	Aplique as alterações na configuração da VM:

```
Update-AzVM -ResourceGroupName $rgName -VM $vm
```

8. Iniciar a VM
	Finalmente, inicie a VM novamente:

```
Start-AzVM -ResourceGroupName $rgName -Name $vmName
```

Aqui está a sequência completa de comandos para redimensionar o disco:

```Powershell
# Install Azure PowerShell module if not already installed
Install-Module -Name Az -AllowClobber -Scope CurrentUser

# Connect to Azure account
Connect-AzAccount

# Select the appropriate subscription
Select-AzSubscription –SubscriptionName 'my-subscription-name'

# Define resource group and VM name
$rgName = 'prod-br-acquirer-rg'
$vmName = 'prod-br-acquirer-sftp-vm'

# Retrieve the VM object
$vm = Get-AzVM -ResourceGroupName $rgName -Name $vmName

# Stop the VM
Stop-AzVM -ResourceGroupName $rgName -Name $vmName

# Resize the disk
$vm.StorageProfile.OSDisk.DiskSizeGB = 200

# Update the VM configuration
Update-AzVM -ResourceGroupName $rgName -VM $vm

# Start the VM
Start-AzVM -ResourceGroupName $rgName -Name $vmName

```

## Atualizar o disco e o FileSystem do Linux

Depois de redimensionar o disco no lado do Azure, siga estas etapas para concluir o processo de redimensionamento na VM do Linux.

1. Verificar a utilização atual do disco
Utilize o comando df para verificar a utilização atual do disco e identificar o dispositivo que necessita de redimensionar:

```bash
df -h
```

Irá perceber que o Data disk agora possui um tamanho maior que antes.

2. Aumentar a partição
Use o comando **growpart** para estender a partição para usar o tamanho total do disco. Substitua **/dev/sdc** (Disco) e **1** (Partição) pelo dispositivo e número de partição apropriados para sua VM:

```bash
growpart /dev/sdc 1
```

Este comando aumenta a partição especificada para preencher o dispositivo subjacente.


> [!WARNING] Importante
> Se atente à "Letra" identificadora do disco. A cada reinicialização essa letra pode mudar, ex.: antes era /dev/sdc e depois passa a ser /dev/sdb.


3. Redimensionar o sistema de arquivos
Use o comando **resize2fs** para redimensionar o sistema de arquivos para corresponder ao novo tamanho da partição. Substitua **/dev/sdb1** e **197G** pelo dispositivo e tamanho apropriados para sua VM:

```bash
resize2fs /dev/sdb1 197G
```

Este comando redimensiona o sistema de arquivos para o tamanho especificado. Observe que o parâmetro de tamanho (197G) deve corresponder ao novo tamanho do seu disco, conforme definido no lado do Azure.

> [!WARNING] Importante
> O valor que indica o tamanho do disco não pode ser o valor total do disco, reduza até que o redimensionamento possa funcionar corretamente.

4. Verificar as alterações
Use o comando **lsblk** para listar informações sobre todos os dispositivos de bloco disponíveis e verificar se as alterações foram aplicadas corretamente:

```
lsblk
```

# Montar o FileShare no FileSystem da VM

Depois que realizar um restart da máquina, pode acontecer de o SFTP gerenciado pela Azure não estar montado no filesystem, apesar de estar configurado em `/etc/fstab`. Siga os passos abaixo:

Digite o comando abaixo para aplicar todos os pontos de montagem existentes no `/etc/fstab`

```bash
mount -a
```

Verifique se o ponto de montagem `//azurefileshare.file.core.windows.net`foi montado.

```
df -h
```

Caso não seja montado, execute manualmente a montagem com o seguinte comando:

```bash
mount -t cifs //azurefileshare.file.core.windows.net/sftp /mnt/prodbracquirersftpvmsa -o credentials=/etc/smbcredentials/azurefileshare.cred,dir_mode=0777,file_mode=0777,serverino,nosharesock,actimeo=30
```

Executando manualmente, poderá debugar se houver algum erro na montagem. Use a flag -v caso for necessário.