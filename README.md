# Deploy a highly available and scalable Wordpress on Azure

This example scenario is a guidance on how to deploy a highly available and scalable Wordpress on Azure using Application Gateway that uses a Virtual Machine Scale Set for backend servers and all deployed into 3 [Availability Zones](https://docs.microsoft.com/en-us/azure/availability-zones/az-overview#availability-zones) to ensure availability and scalability.

Those backend servers will be hosting a [Wordpress](https://wordpress.com/) configured over a NFS Share on [Azure Files](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-introduction) service and connected to an [Azure Database for MySQL](https://docs.microsoft.com/en-us/azure/mysql/overview) service.

To connect the VMs to the MySQL and NFS Share services, two [Azure Private Endpoints](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview) will be configured to connect them privately and securely.

# Architecture

![appgw-wordpress.png](appgw-wordpress.png)

# Prerequisites

* Use the Bash environment in [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/quickstart).

[![launch-cloud-shell.png)](launch-cloud-shell.png)](http://shell.azure.com/)

* If you prefer, [install](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) the Azure CLI to run CLI reference commands.

* This tutorial requires version 2.0.4 or later of the Azure CLI. If using Azure Cloud Shell, the latest version is already installed.

# Define Variables

```
subscriptionId="XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
resourceGroupName="myResourceGroup"
storageAccountName="mystorageacct$RANDOM"
region="westus2"
shareName="myshare$RANDOM"
mysqlServerName="myserver$RANDOM"
mysqlAdmin="myadmin"
mysqlPassword="MyWeaKPassw0rd"
privateEndpointNameStorage="myStoragePrivateEndpoint"
privateConnectionNameStorage="myStorageConnection"
privateDNSZoneNameStorage="privatelink.file.core.windows.net"
privateDNSZoneGroupNameStorage="MyStorageZoneGroup"
privateDNSLinkNameStorage="MyStorageDNSLink"
privateEndpointNameDatabase="myDatabasePrivateEndpoint"
privateConnectionNameDatabase="myDatabaseConnection"
privateDNSZoneNameDatabase="privatelink.mysql.database.azure.com"
privateDNSLinkNameDatabase="MyDatabaseDNSLink"
privateDNSZoneGroupNameDatabase="MyDatabaseZoneGroup"
dbname="wordpressdb"
dbuser="db_user"
dbpassword="db_user-weakPassword"
ScaleSetName="myScaleSet"
VNETName="myVNET"
SubnetName="mySubnet"
BackendSubnetName="myBackendSubnet"
AppGWPublicIPAddressName="myAppGWPublicIP" 
AppGatewayName="myAppGateway"
```
# Create Resource Group
```
az group create --name $resourceGroupName --location $region
```
# Create a VNET
```
az network vnet create \
    --resource-group $resourceGroupName\
    --location $region \
    --name $VNETName \
    --address-prefixes 10.0.0.0/16 \
    --subnet-name $SubnetName  \
    --subnet-prefixes 10.0.0.0/24
```
# Create a Backend Subnet
```
az network vnet subnet create \
  --name $BackendSubnetName \
  --resource-group $resourceGroupName \
  --vnet-name $VNETName \
  --address-prefix 10.0.2.0/24 
 ```
 #  Create a Public IP for the Application Gateway
 ```
 az network public-ip create \
  --resource-group $resourceGroupName \
  --name $AppGWPublicIPAddressName \
  --allocation-method Static \
  --sku Standard \
  --zone 1 2 3
  ```
  # Update the backend subnet  to disable private endpoint network policies for private endpoints
  ```
  az network vnet subnet update \
    --name $BackendSubnetName \
    --resource-group $resourceGroupName \
    --vnet-name $VNETName \
    --disable-private-endpoint-network-policies true
```
# Create the Application Gateway
```
az network application-gateway create \
  --name $AppGatewayName \
  --location $region \
  --resource-group $resourceGroupName \
  --vnet-name $VNETName \
  --subnet $SubnetName \
  --capacity 3 \
  --sku Standard_v2 \
  --http-settings-cookie-based-affinity Enabled \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --public-ip-address $AppGWPublicIPAddressName \
  --zones 1 2 3
  ```
# Create FileStorage Accout
```
az storage account create \
    --resource-group $resourceGroupName \
    --name $storageAccountName \
    --kind FileStorage \
    --sku Premium_ZRS 
```
# Create an NFS share  
```
az storage share-rm create \
    --resource-group $resourceGroupName \
    --storage-account $storageAccountName \
    --name $shareName \
    --enabled-protocol NFS \
    --root-squash NoRootSquash \
    --quota 1024 
```
# Create a Private Endpoint to use with Azure FileStorage
```
idstorage=$(az storage account list \
    --resource-group $resourceGroupName \
    --query '[].[id]' \
    --output tsv)

az network private-endpoint create \
    --name $privateEndpointNameStorage \
    --resource-group $resourceGroupName \
    --vnet-name $VNETName \
    --subnet $BackendSubnetName \
    --private-connection-resource-id $idstorage \
    --connection-name $privateConnectionNameStorage \
    --group-id file
```

