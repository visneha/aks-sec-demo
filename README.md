# aks enterprise security

## Intro

This repo should be considered as a cookbook on how to setup a secure enterprise ready AKS solution. After the rollout your solution will look something like this:

![aks-security-ctx](https://github.com/chrisvugrinec/aks-sec-demo/blob/master/images/aks-secure-ctx.png)

By following the step-by-step cookbook you should be able to rollout the architecture depicted above with the exception of the dotted lines (APIM and App Gateway) which are out of scope for this document.

### Security Boundaries

This document addresses the following security boundaries:

- Accessibility
- Authentication/ Authorisation
- Policies
- Application Security

Accessibility is making the "attack" surface as small as possible. The less exposed you are the harder it will be to be discovered. In order to access this AKS cluster you will need to have network accessibility which is not directly exposed to the internet initially. Mitigation for this implementation:

- AKS private cluster
- Enforce traffic (UDR) to expose only via Azure Firewall
- Disallow creation of public (internet) services on AKS

This cluster is only valid by Persons with a valid credential. Next to that a Non personal account (Service Principal) will be generated for potential build pipelines. There will be a distinction in Administrators (maintainers of the (AKS) infra) and developer (the end users of this cluster).
RBAC will be implemented so that the Personal and Non Personal accounts can only do the allowed operations with the designated namespace within the cluster. No cluster wide roles will be implemented. Mitigations for RBAC and IAM:

- Enable AAD integration
- Enforce RBAC
- Rollout cluster with AAD admin groupID

Policies will be used to enforce governance over the subscription and the underlying AKS clusters and its components. Per AKS cluster a policy enforcer will be deployed which will act as a realtime admission controller to check that the actions are within the policy of the subscription. Initial enabled policies will be:

- enforcement of exposing services only via Internal Loadbalancer
- no allowed usage of elevated containers
- only allow the usage of trusted container repositories

Application Security encompasses features like pod identies, secret management and potential implementations of service meshes. This topic is not in scope for this document.

### AKS features

In order to enable all the aspects, we are enabling the following AKS features/ parameters:

- [enable-rbac](https://docs.microsoft.com/en-us/azure/aks/azure-ad-rbac); configure fine grained AKS rbac. With the aad-admin-group-object-ids param you can define a group ID of Administrators who have elevated K8 rbac rights
- [outbound-type (userDefinedRouting)](https://docs.microsoft.com/en-us/azure/aks/egress-outboundtype); define a predefined route for your AKS traffic
- [enable-pod-security-policy](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-for-kubernetes); enforces realtime policies using an embedded plugin (Gatekeeper) 
- [enable-private-cluster](https://docs.microsoft.com/en-us/azure/aks/private-clusters); your API server is only exposed to the AKS vnet via a private link
- [enable-managed-identity](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity); no more Service Principals in your config. 
- [enable-aad](https://docs.microsoft.com/en-us/azure/aks/azure-ad-integration-cli); nicde integration with Azure Active Directory
  
Please note that some features are still in preview and that there are other excellent solutions available as well you can use in combination or as an alternative to this cookbook.

- [App Gateway Ingress Controller](https://docs.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview); a managed add-on for AKS linking an ingress controller with App Gateway. Ideal for adding policies and reverse proxy rules.
- [API management gateway](https://docs.microsoft.com/en-us/azure/api-management/api-management-kubernetes); ideal for a scenario where you like to have 1 gateway for accessing all your (rest) services. 

### Workflow

The assumption here is that there is a cloud team, responsible for rolling out the (AKS) infrastructure and that there will be a App dev team that will be the end users of AKS.

![aks-security-workflow](https://github.com/chrisvugrinec/aks-sec-demo/blob/master/images/steps.png)

Steps 1 and 2 are executed by the cloud team, step 3 is exectuted by the app dev team using the AKS cluster. The app dev team can use the AKS cluster eiter with their user account or with the provisioned Service Principal, which they can use to setup their deployment pipelines.

No assumptions will be made on CICD tooling, everyhing in the cookbook will be done with basic azure-cli and kubectl commands. Have not tested it on a windows machine as I am using a mac, but it should run on linux/windows as well. In any case you can always use the Integrated cloud shell in the Azure portal.

## Cookbook

### 1. Rollout network infra

#### 1.1 Setup VNETs

Get the sources from this repo: ```git clone https://github.com/chrisvugrinec/aks-sec-demo.git```
change directory: ```cd aks-sec-demo/1_vnet_firewall/1_vnet```
change the parameters in the create.sh file to your likings
execute the shell script: ```./create.sh```

This will create 2 vnets:

- 1 AKS vnet; for hosting all your AKS clusters in different subnets
- 1 Mgmt vnet; for hosting all your shared (infra) services
  
A connection between the 2 networks will be made via vnet peering.

#### 1.2 Create Azure Firewall

change directory: ```cd aks-sec-demo/1_vnet_firewall/2_firewall```
change the parameters in the create.sh file to your likings
execute the shell script: ```./create.sh```

You have created a Public IP address which will be used for external (internet) communication.
You have created a subnet on your management network which is dedicated for hosting your Azure firewall.
You have an Azure Firewall which is configured to have the needed egress communication for your AKS to function. Rules are based on latest documentation.
You have defined a UDR which (once linked to the AKS subnet (later)) will use the firewall as Next Hop for any communication.

When setting up the firewall make sure to be aware of the requirements and limitations stated [here](https://docs.microsoft.com/en-us/azure/aks/limit-egress-traffic).

### 2. Setup bastion

The bastion environment contains vm's that can access (and setup) the AKS environment. To properly set it up enable Just In Time access. The used vnet can also be used in a scenario where you would like to deploy buildagent VM's for your azure devops pipelines.

change directory: ```cd aks-sec-demo/2_bastion```
change the parameters in the create.sh file to your likings
execute the shell script: ```./create.sh```

You can use the ```script_to_base64.txt``` file to create the script you like to be executed when provisioning the VM.
It uses the CustomScript extension for Linux. You need to encode the contents of this script into base64, for eg ```cat script_to_base64.txt | base64``` and copy the base64 code into the ```vm.paramaters.json``` file for the value of the  ```base64-script``` parameter. Once rolled out you should be able to connect to the VM using the Azure portal with the bastion option.

Please note that this distro does not allow password access by default (you need to enable this in the sshd config). You can access it using your public key. If you are using ssh-rsa keys you need to convert it to the openssh standard. Have a look at the ```convertOpenSshToPem.sh``` script.

### 3. Provision AKS

#### 3.1 Create AKS cluster

change directory: ```cd aks-sec-demo/3_aks```
change the parameters in the 1_create.sh file to your likings
execute the shell script: ```./1_create.sh```

This will 1st create an AKS subnet within the earlier defines AKS vnet. 
Please make sure you configure a subnet CIDR in the param for this AKS subnet that is within the range of the AKS vnet.
The Subnet will be linked to the earlier defined RouteTable.
Finally it will create the AKS cluster.  

If you would like a group of administrator to have elevated rights then change the script to use the aks-with-admin.json template and configure this param: 

```
                "aadProfile": {
                    "adminGroupObjectIDs": [
                        "[parameters('adminGroupId')]"
                    ],
                },
```

the adminGroupId is the GroupId you have to have configured in Azure AD. You can create the group and populate the groupmembers within the Azure Portal.

In this phase of the cookbook you will have 3 new resourcegroups:

- 1 resourcegroup containing all your admin (shared) resources
- 1 resourcegroup which contains the azure resources needed for your appdev team, containing the AKS cluster for example
- 1 resourcegroup containing all the AKS resources, MC_[ RESOURCEGROUPNAME ]_[ CLUSTERNAME ] . This is the default naming it is possible to specify this differently.

#### 3.2 Create DNS stuff

change the parameters in the create.sh file to your likings
execute the shell script: ```./2_createDNSstuff.sh```

This script will do 2 things:

*Expose fqdn of API to mgmt vnet*

When the AKS private cluster is made a virtual NIC of the admin server(s) is exposed in the AKS subnet (see MC resources). The AKS cluster is by default aware of it as a DNS zone has been created exposing the fqdn of the API within the AKS subnet. The 1st part of the script uses that DNS zone and creates a dns Link to the management VNET, so resources within that VNET are also aware of the fqdn of the API server of the new AKS cluster.

*Create a DNS zone for appdev purposes*

Secondly within the appdev resourcegroup a DNS zone is created with an entry [clustername].[defined zone name].
This entry is linked to an IP address that will be linked to the Internal Loadbalancer (to be created in next step).

Please make sure when you configure a loadbalancer IP that this IP is within the range of the AKS subnet.

#### 3.3 Assign rights and roles

First start off with creating a group in Azure AD that contains users of your AKS cluster.

![aad1](https://github.com/chrisvugrinec/aks-sec-demo/blob/master/images/aad1.png)

Define a group; you can also script/automate this if you have sufficient rights on your tenant.

![aad2](https://github.com/chrisvugrinec/aks-sec-demo/blob/master/images/aad1.png)


change the parameters in the create.sh file to your likings
execute the shell script: ```./3_assign_RightsAndRoles.sh```








