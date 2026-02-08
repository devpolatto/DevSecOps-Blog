---
title: Daily commands
tags:
  - Azure
enableToc: true
---

# Azure CLI

# IAM

```bash
az role assignment create \
  --assignee 3693889f-c718-4bce-bfa0-2a9b31e342eb \
  --role Contributor \
  --scope "/subscriptions/b57d722b-40b5-4749-b9f6-71be54d0a0c4/resourceGroups/prod-merchant-portal"
```

# Disk

```
az disk list --resource-group dev-westus2-sftp-poc-unmanaged-disk-rg

az disk show --resource-group dev-westus2-sftp-poc-unmanaged-disk-rg --disk-name dev-westus2-sftp-poc-unmanaged-disk-sftp-vm_osd_853055a4e91c4800986c68604b70ce5f

az disk show --resource-group dev-westus2-sftp-poc-unmanaged-disk-rg --disk-name dev-westus2-sftp-poc-unmanaged-disk-sftp-vm_dev_ddd3ad22518948969939fc8d90f728d2
```

# Network

## Subnet

```
az network vnet subnet show \
  --resource-group prod-useus2-IPCloud \
  --vnet-name prod-useus2-IPCloud-vnet \
  --name prod-useus2-IPCloud-snet
```

## Public IP

```shell
az network public-ip list \
--subscription $ACQIO_PROD \
--query "[?allocationMethod=='Static'].{Name:name, ResourceGroup:resourceGroup, Location:location, IPAddress:ipAddress, SKU:sku.name}" -o table


az network public-ip list \
--subscription $ACQIO_PROD \
--query "[?sku.name=='Standard']" \
-o table

az network public-ip list \
--subscription $ACQIO_PROD \
--query "[?sku.name=='Basic'].{Name:name, ResourceGroup:resourceGroup, AllocationMethod:publicIPAllocationMethod, SKU:sku.name}" \
-o table

az network public-ip show \
--subscription $ACQIO_PROD \
--resource-group vpn-gateway \
--name vpn-gateway-ip
```

# VPN Gateway

```shell
az network vnet-gateway list \
  --subscription $ACQIO_PROD_GATEWAY \
  --resource-group vpn-gateway \
  --query "{IP_Gateway:[].ipConfigurations[].name}" \
  -o table
  
az network vnet-gateway show \
--subscription $ACQIO_PROD_GATEWAY \
--resource-group vpn-gateway \
--name prodgtw-vpn-gateway

az network vnet-gateway show \
--subscription $ACQIO_PROD \
--resource-group vpn-gateway \
--name prod-vpn-gateway \
--query "bgpSettings.bgpPeeringAddresses[].{BGP:defaultBgpIpAddresses[0],Tunnel:tunnelIpAddresses[0]}" \
-o table

az network vnet-gateway list \
  --subscription $ACQIO_PROD_GATEWAY \
  --resource-group vpn-gateway \
  --query "[].ipConfigurations[].name" \
  -o table
```

## Connections

```shell
az network vpn-connection list \
--subscription $ACQIO_PROD_GATEWAY \
--resource-group vpn-gateway

az network vpn-connection list \
--subscription $ACQIO_DEV \
--resource-group vpn-gateway \
--query "[].{Name:name, ResourceGroup:resourceGroup}" \
-o table

az network vpn-connection show \
--subscription $ACQIO_PROD \
--resource-group vpn-gateway \
--name 2rp \
--query "{Name:name, Status:connectionStatus, LocalNetwork:trafficSelectorPolicies[0].localAddressRanges}" \
-o table

az network vpn-connection show \
  --subscription $ACQIO_PROD_GATEWAY \
  --resource-group vpn-gateway \
  --name redsys_fs \
  --query "{Tunnel:tunnelConnectionStatus[0].tunnel, Status:connectionStatus, LocalSubnet:trafficSelectorPolicies[0].localAddressRanges[0], RemoteSubnet:trafficSelectorPolicies[0].remoteAddressRanges[0]}" \
  -o table
```


```shell  
az network vpn-connection create \
    --subscription $ACQIO_PROD \
    --name 2rp \
    --resource-group vpn-gateway \
    --location westus2 \
    --vnet-gateway1 prod-vpn-gateway \
    --local-gateway2 2rp \
    --shared-key "5OoO@VhmSStSB5USkGq8@0U@YyLTjF%t" \
    --enable-bgp false \
    --express-route-gateway-bypass false \
    --routing-weight 0 \
    --use-policy-based-traffic-selectors false \
    --tags environment=prod repository=spokane repository_path=infra/terraform/projects/vpn-gateway resource_group=vpn-gateway squad-team=devops subscription=prod-engineering terraform=true tribe=shared
    
az network vpn-connection ipsec-policy add \
    --subscription $ACQIO_PROD \
    --connection-name 2rp \
    --resource-group vpn-gateway \
    --dh-group DHGroup14 \
    --ike-encryption AES256 \
    --ike-integrity SHA256 \
    --ipsec-encryption AES256 \
    --ipsec-integrity SHA256 \
    --pfs-group PFS2048 \
    --sa-lifetime 3600 \
    --sa-max-size 0
    
az network vpn-connection update \
    --subscription $ACQIO_PROD \
    --name 2rp \
    --resource-group vpn-gateway \
    --set dpdTimeoutSeconds=10 \
    --set connectionMode="Default" \
    --set connectionProtocol="IKEv2" \
    --set useLocalAzureIpAddress=false \
    --set enablePrivateLinkFastPath=false
```

## Local Network gateway

```shell
az network local-gateway show \
  --subscription $ACQIO_PROD_GATEWAY \
  --resource-group vpn-gateway \
  --name redsys_fs \
  --query "{RemoteGatewayIPAddress:gatewayIpAddress, RemoteSubnet:localNetworkAddressSpace.addressPrefixes}" \
  -o json

az network local-gateway show \
  --subscription $ACQIO_PROD \
  --resource-group vpn-gateway \
  --name emprel \
  --query "{RemoteGatewayIPAddress:gatewayIpAddress, RemoteSubnet:localNetworkAddressSpace.addressPrefixes}" \
  -o table
```

## Routes

```shell
az network vnet-gateway get-routes-information \
--subscription $ACQIO_PROD \
--resource-group vpn-gateway \
--name prod-vpn-gateway

az network vnet-gateway list-advertised-routes \
--subscription $ACQIO_PROD \
--resource-group vpn-gateway \
--name prod-vpn-gateway \
--peer 10.254.254.4

az network vnet-gateway list-learned-routes \
  --subscription $ACQIO_PROD \
  --resource-group vpn-gateway \
  --name prod-vpn-gateway \
  --query "value[?network=='172.21.0.0/18' || network=='192.168.0.0/16'].
    {
      Network: network,
      LocalAddress: localAddress,
      SourcePeer: sourcePeer,
      Origin: origin,
      NextHop: nextHop
    }" \
  -o table
```

# Storage Account

```bash
az storage account show \
--subscription $ACQIO_PROD \
--name prodbracquirersftpvmsa \
--resource-group prod-br-acquirer-rg
```

## Blob

```bash
az storage blob upload-batch \
-s dist \
-d \$web \
--account-name prodmerchantportalnv \
--subscription b57d722b-40b5-4749-b9f6-71be54d0a0c4 \
--overwrite true
```
## Files

```bash
az storage file list \
--account-name prodbracquirersftpvmsa \
--share-namec sftp
```

### Local user


```shell
az storage account local-user list-keys \
--subscription $ACQIO_TEST \
--resource-group dev-test-uswe2-Private-endpoint \
--name sftpuser \
--account-name testsftpstorage2025 \
--query sshPassword
```

```bash
az storage account local-user show \
--subscription $ACQIO_TEST \
--resource-group dev-test-uswe2-Private-endpoint \
--account-name testsftpstorage2025 \
--name sftpuser
```
# Frontdoor

## Profile

```bash
az afd profile list \
--subscription $ACQIO_PROD \
--resource-group prod-merchant-portal
```

## Endpoint

```bash
az afd endpoint list \
--subscription $ACQIO_PROD \
--resource-group prod-merchant-portal \
--profile-name prodmerchantportalnew
```

## Route

```bash
az afd route list \
--subscription $ACQIO_PROD \
--resource-group prod-merchant-portal \
--endpoint-name prodmerchantportalnew \
--profile-name prodmerchantportalnew
```

## Purge Cache CDN

```bash
az afd endpoint purge \
--subscription b57d722b-40b5-4749-b9f6-71be54d0a0c4 \
--resource-group prod-merchant-portal \
--profile-name prodmerchantportal \
--endpoint-name prodmerchantportal \
--content-paths '/*'
```

# PostgreSQL flexible-server

```bash
az postgres flexible-server parameter set \
  --resource-group <your-resource-group> \
  --server-name prod-wus2-datahub-internal-dbfs-pgdb-new \
  --name azure.extensions \
  --value "pg_buffercache,pg_stat_statements" \
  --subscription <your-subscription-id>
```

# SQL Server

```shell
az sql server firewall-rule create \
--subscription $ACQIO_PRODBACK \
--resource-group prod-br-tpv-rg \
--server  \
--name prod-br-tpv-acqio \
--start-ip-address 45.235.94.151 \
--end-ip-address 45.235.94.151
```

# DNS

## record-set

### A

```shell
az network dns record-set a add-record \
--subscription $ACQIO_PROD \
--resource-group prod-global-dns-rg \
--zone-name acqio.com.br \
--record-set-name lojista \
--ttl 3600 \
--ipv4-address 20.252.57.64

az network dns record-set a delete \
--subscription $ACQIO_PROD \
--resource-group prod-global-dns-rg \
--zone-name acqio.com.br \
--name lojista
```

### CNAME

```shell
az network dns record-set cname show \
--subscription $ACQIO_PROD \
--resource-group prod-global-dns-rg \
--zone-name acqio.com.br \
--name lojista

az network dns record-set cname delete \
--subscription $ACQIO_PROD \
--resource-group prod-global-dns-rg \
--zone-name acqio.com.br \
--name lojista

az network dns record-set cname create \
--subscription $ACQIO_PROD \
--resource-group prod-global-dns-rg \
--zone-name acqio.com.br \
--name lojista \
--target-resource "/subscriptions/b57d722b-40b5-4749-b9f6-71be54d0a0c4/resourceGroups/prod-merchant-portal/providers/Microsoft.Cdn/profiles/prodmerchantportalnew/afdendpoints/prodmerchantportalnew" \
--ttl 3600
```



# Azure Query
## Verifica discos não vinculados a uma VM

```sql
resources | where type =~ 'microsoft.compute/disks' and managedBy == ''| extend diskState = tostring(properties.diskState)| where managedBy == '' and diskState != 'ActiveSAS'or diskState == 'Unattached' and diskState != 'ActiveSAS' and tags !contains 'ASR-ReplicaDisk' and tags !contains 'asrseeddisk'| project subscriptionId,id
```