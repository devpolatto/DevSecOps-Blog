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
  --assignee <user-id> \
  --role Contributor \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>"
```

# Disk

```
az disk list --resource-group <resource-group>

az disk show --resource-group <resource-group> --disk-name <disk-name>
```

# Network

## Subnet

```shell
az network vnet subnet show \
  --resource-group <resource-group> \
  --vnet-name <vnet-name> \
  --name <subnet-name>
```

## Public IP

```shell
az network public-ip list \
--subscription <subscription-id> \
--query "[?allocationMethod=='Static'].{Name:name, ResourceGroup:resourceGroup, Location:location, IPAddress:ipAddress, SKU:sku.name}" -o table


az network public-ip list \
--subscription $<subscription-id> \
--query "[?sku.name=='Standard']" \
-o table

az network public-ip list \
--subscription $<subscription-id> \
--query "[?sku.name=='Basic'].{Name:name, ResourceGroup:resourceGroup, AllocationMethod:publicIPAllocationMethod, SKU:sku.name}" \
-o table

az network public-ip show \
--subscription <subscription-id> \
--resource-group <resource-group> \
--name <public-ip-name>
```

# VPN Gateway

```shell
az network vnet-gateway list \
  --subscription <subscription-id> \
  --resource-group <resource-group> \
  --query "{IP_Gateway:[].ipConfigurations[].name}" \
  -o table
  
az network vnet-gateway show \
--subscription <subscription-id> \
--resource-group <resource-group> \
--name <gateway-name>

az network vnet-gateway show \
--subscription $<subscription-id> \
--resource-group vpn-gateway \
--name prod-vpn-gateway \
--query "bgpSettings.bgpPeeringAddresses[].{BGP:defaultBgpIpAddresses[0],Tunnel:tunnelIpAddresses[0]}" \
-o table

az network vnet-gateway list \
  --subscription $<subscription-id>_GATEWAY \
  --resource-group vpn-gateway \
  --query "[].ipConfigurations[].name" \
  -o table
```

## Connections

```shell
az network vpn-connection list \
--subscription $<subscription-id>_GATEWAY \
--resource-group vpn-gateway

az network vpn-connection list \
--subscription <subscription-id> \
--resource-group vpn-gateway \
--query "[].{Name:name, ResourceGroup:resourceGroup}" \
-o table

az network vpn-connection show \
--subscription $<subscription-id> \
--resource-group vpn-gateway \
--name 2rp \
--query "{Name:name, Status:connectionStatus, LocalNetwork:trafficSelectorPolicies[0].localAddressRanges}" \
-o table

az network vpn-connection show \
  --subscription $<subscription-id>_GATEWAY \
  --resource-group vpn-gateway \
  --name redsys_fs \
  --query "{Tunnel:tunnelConnectionStatus[0].tunnel, Status:connectionStatus, LocalSubnet:trafficSelectorPolicies[0].localAddressRanges[0], RemoteSubnet:trafficSelectorPolicies[0].remoteAddressRanges[0]}" \
  -o table
```


```shell  
az network vpn-connection create \
  --subscription <subscription-id> \
  --name <connection-name> \
  --resource-group <resource-group> \
  --location <location> \
  --vnet-gateway1 <vnet-gateway-name> \
  --local-gateway2 <local-gateway-name> \
  --shared-key <shared-key> \
  --enable-bgp false \
  --express-route-gateway-bypass false \
  --routing-weight 0 \
  --use-policy-based-traffic-selectors false \
  --tags environment=prod repository=repo repository_path=infra/terraform/projects/vpn-gateway resource_group=vpn-gateway squad-team=devops subscription=prod-engineering terraform=true tribe=shared
    
az network vpn-connection ipsec-policy add \
    --subscription $<subscription-id> \
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
    --subscription $<subscription-id> \
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
  --subscription $<subscription-id>_GATEWAY \
  --resource-group vpn-gateway \
  --name redsys_fs \
  --query "{RemoteGatewayIPAddress:gatewayIpAddress, RemoteSubnet:localNetworkAddressSpace.addressPrefixes}" \
  -o json

az network local-gateway show \
  --subscription $<subscription-id> \
  --resource-group vpn-gateway \
  --name emprel \
  --query "{RemoteGatewayIPAddress:gatewayIpAddress, RemoteSubnet:localNetworkAddressSpace.addressPrefixes}" \
  -o table
```

## Routes

```shell
az network vnet-gateway get-routes-information \
--subscription $<subscription-id> \
--resource-group vpn-gateway \
--name prod-vpn-gateway

az network vnet-gateway list-advertised-routes \
--subscription $<subscription-id> \
--resource-group vpn-gateway \
--name prod-vpn-gateway \
--peer 10.254.254.4

az network vnet-gateway list-learned-routes \
  --subscription $<subscription-id> \
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
--subscription <subscription-id> \
--name <storage-account-name> \
--resource-group <resource-group>
```

## Blob

```bash
az storage blob upload-batch \
-s dist \
-d \$web \
--account-name <storage-account-name> \
--subscription <subscription-id> \
--overwrite true
```
## Files

```bash
az storage file list \
--account-name <storage-account-name> \
--share-name <share-name>
```

### Local user


```shell
az storage account local-user list-keys \
--subscription <subscription-id> \
--resource-group <resource-group> \
--name <local-user-name> \
--account-name <storage-account-name> \
--query sshPassword
```

```bash
az storage account local-user show \
--subscription <subscription-id> \
--resource-group <resource-group> \
--account-name <storage-account-name> \
--name <local-user-name>
```
# Frontdoor

## Profile

```bash
az afd profile list \
--subscription <subscription-id> \
--resource-group <resource-group>
```

## Endpoint

```bash
az afd endpoint list \
--subscription <subscription-id> \
--resource-group <resource-group> \
--profile-name <profile-name>
```

## Route

```bash
az afd route list \
--subscription <subscription-id> \
--resource-group <resource-group> \
--endpoint-name <endpoint-name> \
--profile-name <profile-name>
```

## Purge Cache CDN

```bash
az afd endpoint purge \
--subscription <subscription-id> \
--resource-group <resource-group> \
--profile-name <profile-name> \
--endpoint-name <endpoint-name> \
--content-paths '/*'
```

# PostgreSQL flexible-server

```bash
az postgres flexible-server parameter set \
  --resource-group <resource-group> \
  --server-name <server-name> \
  --name azure.extensions \
  --value "pg_buffercache,pg_stat_statements" \
  --subscription <subscription-id>
```

# SQL Server

```shell
az sql server firewall-rule create \
--subscription <subscription-id> \
--resource-group <resource-group> \
--server <server-name> \
--name <firewall-rule-name> \
--start-ip-address <ip-address> \
--end-ip-address <ip-address>
```

# DNS

## record-set

### A

```shell
az network dns record-set a add-record \
--subscription <subscription-id> \
--resource-group <resource-group> \
--zone-name <zone-name> \
--record-set-name <record-set-name> \
--ttl 3600 \
--ipv4-address <ip-address>

az network dns record-set a delete \
--subscription <subscription-id> \
--resource-group <resource-group> \
--zone-name <zone-name> \
--name <record-set-name>
```

### CNAME

```shell
az network dns record-set cname show \
--subscription <subscription-id> \
--resource-group <resource-group> \
--zone-name <zone-name> \
--name <record-set-name>

az network dns record-set cname delete \
--subscription <subscription-id> \
--resource-group <resource-group> \
--zone-name <zone-name> \
--name <record-set-name>

az network dns record-set cname create \
--subscription <subscription-id> \
--resource-group <resource-group> \
--zone-name <zone-name> \
--name <record-set-name> \
--target-resource "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Cdn/profiles/<profile-name>/afdendpoints/<endpoint-name>" \
--ttl 3600
```

# AKS

```shell
az aks show \
--subscription <subscription-id> \
  --resource-group <resource-group> \
  --name <aks-name> \
  --query "networkProfile.loadBalancerProfile.effectiveOutboundIPs[].id" -o tsv
```

# Azure Query
## Verifica discos não vinculados a uma VM

```sql
resources | where type =~ 'microsoft.compute/disks' and managedBy == ''| extend diskState = tostring(properties.diskState)| where managedBy == '' and diskState != 'ActiveSAS'or diskState == 'Unattached' and diskState != 'ActiveSAS' and tags !contains 'ASR-ReplicaDisk' and tags !contains 'asrseeddisk'| project subscriptionId,id
```