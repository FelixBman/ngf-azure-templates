# Barracuda CloudGen Firewall for Azure - Custom High Availability Cluster with dual NIC 

## Introduction

Traditionally, Barracuda CloudGen Firewall (CGF) uses UDR rewriting technique to redirect traffic when an HA failover happens. This method works well for smaller deployments, but has few drawbacks when using peered VNets or if corporate policy restricts saving AAD authentication keys in 3rd party software configuration.

Azure ILB solves above problems, providing failover capabilities with zero integration with the cloud fabric, offers shorter failover times (~15 seconds) independent of network complexity, and provides stateful failover.

This template deploys a pair of Firewalls, an Internal and External Standard Load Balancer into a existing VNET and subnets. It expects a frontend and backend subnet to exist. 

![Network diagram](images/cgf-ha-2nic-elb-ilb.png)

## Prerequisites

The solution does a check of the template when you use the provided scripts. It does require that [Programmatic Deployment](https://azure.microsoft.com/en-us/blog/working-with-marketplace-images-on-azure-resource-manager/) is enabled for the Barracuda Next Gen Firewall BYOL or PAYG images. Barracuda recommends use of **D**, **D_v2**, **F** or newer series. 

You can enable programatic deployment via Powershell using the Cloud Shell feature in the portal. Below are two powershell examples for byol and hourly, please adapt as required to your version of powershell and byol or hourly license requirement.

`Get-AzMarketplaceTerms -Publisher "barracudanetworks" -Product "barracuda-ng-firewall" -Name "byol" | Set-AzMarketplaceTerms -Accept`
`Get-AzureRmMarketplaceTerms -Publisher "barracudanetworks" -Product "barracuda-ng-firewall" -Name "hourly" | Set-AzureRmMarketplaceTerms -Accept`

## Deployment

The package provides a deploy.ps1 and deploy.sh for Powershell or Azure CLI based deployments. This can be peformed from the Azure Portal as well as the any system that has either of these scripting infrastructures installed. Or you can deploy from the Azure Portal using the provided link.

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fbarracudanetworks%2Fngf-azure-templates%2Fmaster%2Fcontrib%2FCGF-Custom-HA-2NIC-AS-ELB-ILB-STD%2Fazuredeploy.json" target="_blank"><img src="http://azuredeploy.net/deploybutton.png"/></a>
<a href="http://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2Fbarracudanetworks%2Fngf-azure-templates%2Fmaster%2Fcontrib%2FCGF-Custom-HA-2NIC-AS-ELB-ILB-STD%2Fazuredeploy.json" target="_blank">
    <img src="http://armviz.io/visualizebutton.png"/>
</a>

## Deployed resources
Following resources will be created by the template:
- An availabilty set containing both firewalls
- One internal standard Azure Load Balancer as the default gateway for all traffic that needs inspection
- One external standard Azure Load Balancer containing the deployed virtual machines with a public IP and services for IPSEC and TINA VPN tunnels available
- Two Barracuda CloudGen Firewall virtual machines with 1 network interface each and public IP


**Note** Additional backend subnets and resources are *not* automatically created by the template. This has to be done manually after template deployment has finished or by adapting the ARM template.

## Next Steps

After successful deployment you can manage them using CloudGen Admin application available from Barracuda Download Portal. Management IP addresses you'll find in firewall instances properties, username is *root* and the password is what you provided during template deployment.

## Post Deployment Configuration

### Networking

To activate the secondary interface you need to following certain steps. Also additional routing in the firewall is required to route the packets in the correct direction. 

Switch the config to Advanced Mode. 


- Firewall Admin > Configuration > Network > IP configuration and switch the config to Advanced mode. 
![Network diagram](images/enabledadvancedmode.png)

- Firewall Admin > Configuration > Network > Additional Local IP's and add the private IP of the internal NIC in here linked to eth1. <br/>
![Network diagram](images/cgf-ha-2nic-network3.png) <br/>
![Network diagram](images/cgf-ha-2nic-network4.png)

- Firewall Admin > Configuration > Network > Advanced Routing add the routes for the Internal network ranges towards the default gateway of the internal firewall subnet (172.16.136.129).<br/>
![Network diagram](images/cgf-ha-2nic-network5.png)

- Firewall Admin > Configuration > Network > Routing add the source based routing for magic IP from Microsoft Azure that is used for the health probe of the Load Balancer. This ensures that probes towards the internal NIC will be responded by the internal NIC. <br/>
![Network diagram](images/cgf-ha-2nic-network7.png) <br/>
![Network diagram](images/cgf-ha-2nic-network8.png)

- Firewall Admin > Control > Box > Activate new network configuration to enable all the network changes on the network layer of the firewall <br/>
![Network diagram](images/cgf-ha-2nic-network9.png) <br/>
![Network diagram](images/cgf-ha-2nic-network10.png) <br/>

### Internal Load Balancer healthprobe

You need to enable the firewall that allows the load balancer health probes to function. Go to Configuration > Assigned Services > NGFW > Forwarding Rules 
Lock the config and right click and edit. 

Modify as below and activate the rule. 

![Example firewall probe redirection rule](images/modifiedrule.png)

For more information on App Redirect rule consult Barracuda Campus: [How to Create an App Redirect Access Rule](https://campus.barracuda.com/product/cloudgenfirewall/doc/96026195/)

It is also recommended you harden management access by enabling multifactor or key authentication and by restricting access to management interface using Management ACL: [How to Change the Root Password and Management ACL](https://campus.barracuda.com/product/cloudgenfirewall/doc/96026575/)

## Template Parameters
| Parameter Name | Description
|---|---
adminPassword | Password for the CloudGen Firewall Admin tool 
prefix | identifying prefix for all VM's being build. e.g WeProd would become WeProd-VM-CGF (Max 19 char, no spaces, [A-Za-z0-9]
vNetResourceGroup | Name of the resource group the VNET is in
vNetName | Name of the VNET
frontendSubnetName | Name of the frontend subnet
backendSubnetName | Name of the backend subnet
frontendSubnetRange | Network range of the red subnet (e.g. 172.16.136.0/25)
backendSubnetRange | Network range of the green subnet (e.g. 172.16.136.128/25)
imageSKU | SKU Hourly (PAYG) or BYOL (Bring your own license)
vmSize | Size of the VMs to be created
enableAccelerated | yes or no to enable accelerated networking support
ccManaged | Is this instance managed via a CloudGen Control Center (Yes/No)
ccClusterName | The name of the cluster of this instance in the CloudGen Control Center
ccRangeId | The range location of this instance in the CloudGen Control Center
ccIpAddress | IP address of the CloudGen Control Center
ccSecret | Secret to retrieve the configuration from the CloudGen Control Center
