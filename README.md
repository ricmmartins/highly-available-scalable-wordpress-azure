# Deploy a highly available and scalable Wordpress on Azure

This example scenario is a guidance on how to deploy a highly available and scalable Wordpress on Azure using Application Gateway that uses a Virtual Machine Scale Set for backend servers and all deployed into 3 [Availability Zones](https://docs.microsoft.com/en-us/azure/availability-zones/az-overview#availability-zones) to ensure availability and scalability.

Those backend servers will be hosting a [Wordpress](https://wordpress.com/) configured over a NFS Share on [Azure Files](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-introduction) service and connected to an [Azure Database for MySQL](https://docs.microsoft.com/en-us/azure/mysql/overview) service.

To connect the VMs to the MySQL and NFS Share services, two [Azure Private Endpoints](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview) will be configured to connect them privately and securely.

# Architecture

![appgw-wordpress.png](appgw-wordpress.png)

