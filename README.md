# Projeto Jenkins (Monitoramento de Infraestrutura) 

### Premissas

1 - Subscrição na azure <br />
2 - Principal Criado no azure com contribuidor ativo (abaixo procedimento de criação) <br />
3 - Azure DevOps <br />
4 - Connection com Azure DevOps <a href="https://dev.to/truelime/how-to-connect-the-azure-devops-release-pipeline-and-deploy-to-azure-using-a-service-connection-and-azure-app-registration-3566#:~:text=How%20to%20connect%20the%20Azure%20DevOps%20Release%20pipeline,5%20Configure%20a%20new%20Azure%20service%20connection%20"> link de exemplo </a>.    <br />
5 - Storage Account com FIle Server Criado, o mesmo será montado no diretorio "/jobs" do servidor jenkins para publicacao das pipeline dos servidores servidores em HA (Para fins de estudo abstraimos o passo de criacao do storage account, forma de criacao esta disponivel atraves do link  <br />


<a href="https://learn.microsoft.com/en-us/azure/storage/files/storage-how-to-use-files-portal?tabs=azure-portal"> Create and use an Azure file share </a>. 

## Passo 1 Ex: Criação  de usuário 
Para Criação do princiapal que será utilizado no conecction do azure devops é necessário realizar o comando abaixo 

az ad sp create-for-rbac --name MyPipeline --role Contributor --scopes "/subscriptions/2c81448w21********"

![image](https://user-images.githubusercontent.com/28166733/227680438-fcf9c9a9-38f7-4045-bc9e-6e19af5625a6.png)

## Passo 2

### Após copiar os valores gerados acima para um bloco de notas, é necessário criar um novo projeto no azure devops para que possamos rodar as nossas pipelines de automação via IaC, 


<img width="416" alt="image" src="https://user-images.githubusercontent.com/28166733/221380592-00e13aa1-c3f8-4341-9f07-20c771f2a96b.png">


Após criar o projeto, va até project settings e aba o menu de Service Connections, clique em create service connection 

<img width="827" alt="image" src="https://user-images.githubusercontent.com/28166733/221380670-e6aecbeb-5cea-4568-bbd7-a1230ed5a81f.png">

Vá em Azure Resource Manager e clique em next, 

<img width="326" alt="image" src="https://user-images.githubusercontent.com/28166733/221380860-440a9768-f4a1-466f-b446-d06893b7ceb0.png">

Vá em service principal manual e preencha com as informações salvas no bloco de notas que realizamos no passo 1 

<img width="347" alt="image" src="https://user-images.githubusercontent.com/28166733/221380891-a4014054-de41-452a-9614-8b83308222de.png">

Após preencher todas as informações com o Subscription Name, Tenant ID, Subscription ID, Princiapl ID, Serivce Key você poderá clicar em <img width="291" alt="image" src="https://user-images.githubusercontent.com/28166733/221381566-fb4b1090-6e62-4617-b315-f9d951097577.png">
 
clique em Verify para verificar se foi possível realizar a conexão


<img width="239" alt="image" src="https://user-images.githubusercontent.com/28166733/221381220-54b98f02-5984-4067-bf79-7f80f95603ef.png">

Criamos a conexão chamada MyPipelineDevOps

<img width="318" alt="image" src="https://user-images.githubusercontent.com/28166733/221424629-61575d4b-2faa-45fa-8e90-e72871a370b1.png">


após conexão criada com sucesso, retorne para o respositório MBA Criado e crie um novo folder e nomeie como ARM 

![image](https://user-images.githubusercontent.com/28166733/221381383-b6dfd719-79ab-40ca-b1c5-9a22d62fe704.png)


***

# Criacao do IaC
Criaremos 6 arquivos  conforme figura abaixo

![image](https://user-images.githubusercontent.com/28166733/227672352-046db9a6-6c7e-4bd2-9ab0-b65d02c96733.png)


* Arquivo responsável pela criação de 2 VMs Jenkins e 1 load balancer e 1 arquivo SH para instalacao e configuracao do jenkins: </br> 
 _arm/jenkinsnodes.bicep_  </br>
 _arm/jenkins.sh_ </br> 
* Arquivo responsável pela criação do nodea que rodará uma instância docker com 1 container prometheus e 1 container grafana: </br> 
 _arm/monitoringnodea.bicep_  </br>
 _arm/monitoringnodea.sh_ </br> 
* Arquivo responsável pela criação do nodea que rodará uma instância docker com 1 container prometheus e 1 container grafana:   
 _arm/monitoringnodeb.bicep_  </br> 
 _arm/monitoringnodea.sh_
* Arquivo de pipeline do Azure Pipelines : </br> 
 _azure-pipelines.yml_ 

## Arquivo jenkinsnodes.bicep
Crie os arquivos chamado  _arm/jenkinsnodes.bicep_ e ensira os códigos abaixo

```bicep
@description('Specifies a project name that is used for generating resource names.')
param projectName string

@description('Specifies the location for all of the resources created by this template.')
param location string = resourceGroup().location

@description('Specifies the virtual machine administrator username.')
param adminUsername string

@description('Specifies the virtual machine administrator password.')
@secure()
param adminPassword string

@description('Size of the virtual machine')
param vmSize string = 'Standard_B1ms'

@description('osversion')
param ubuntuOSVersion string = 'Ubuntu-2004'

@description('password or key')
param authenticationType string = 'password'
var lbName = '${projectName}-lb'
var lbSkuName = 'Standard'
var lbPublicIpAddressName = '${projectName}-lbPublicIP'
var lbPublicIPAddressNameOutbound = '${projectName}-lbPublicIPOutbound'
var lbFrontEndName = 'LoadBalancerFrontEnd'
var lbFrontEndNameOutbound = 'LoadBalancerFrontEndOutbound'
var lbBackendPoolName = 'LoadBalancerBackEndPool'
var lbBackendPoolNameOutbound = 'LoadBalancerBackEndPoolOutbound'
var lbProbeName = 'loadBalancerHealthProbe'
var nsgName = '${projectName}-nsg'
var vNetName = '${projectName}-vnet'
var vNetAddressPrefix = '10.0.0.0/16'
var vNetSubnetName = 'BackendSubnet'
var vNetSubnetAddressPrefix = '10.0.0.0/24'
var osDiskType = 'Standard_LRS'
var imageReference = {
  'Ubuntu-1804': {
    publisher: 'Canonical'
    offer: 'UbuntuServer'
    sku: '18_04-lts-gen2'
    version: 'latest'
  }
  'Ubuntu-2004': {
    publisher: 'Canonical'
    offer: '0001-com-ubuntu-server-focal'
    sku: '20_04-lts-gen2'
    version: 'latest'
  }
  'Ubuntu-2204': {
    publisher: 'Canonical'
    offer: '0001-com-ubuntu-server-jammy'
    sku: '22_04-lts-gen2'
    version: 'latest'
  }
}

var linuxConfiguration = {
  disablePasswordAuthentication: false
  ssh: {
    publicKeys: [
      {
        path: '/home/${adminUsername}/.ssh/authorized_keys'
        keyData: adminPassword
      }
    ]
  }
}


resource project_vm_1_networkInterface 'Microsoft.Network/networkInterfaces@2021-08-01' = [for i in range(0, 2): {
  name: '${projectName}-vm${(i + 1)}-networkInterface'
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          privateIPAllocationMethod: 'Dynamic' 
        }
          subnet: {
            id: vNetName_vNetSubnetName.id
          }
          loadBalancerBackendAddressPools: [
            {
              id: resourceId('Microsoft.Network/loadBalancers/backendAddressPools', lbName, lbBackendPoolName)
            }
            {
              id: resourceId('Microsoft.Network/loadBalancers/backendAddressPools', lbName, lbBackendPoolNameOutbound)
            }
          ]
        }
      }
    ]
    networkSecurityGroup: {
      id: nsg.id
    }
  }
  dependsOn: [
    vNet
    lb
  ]
}]



resource project_vm_1 'Microsoft.Compute/virtualMachines@2021-11-01' = [for i in range(1, 2): {
  name: '${projectName}-vm${i}'
  location: location
  zones: [
    string(i)
  ]
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    storageProfile: {
      osDisk: {
        
        createOption: 'FromImage'
        managedDisk: {
          storageAccountType: osDiskType
        }
      }
      imageReference: imageReference[ubuntuOSVersion]
    }

    networkProfile: {
      networkInterfaces: [
        {
          id: resourceId('Microsoft.Network/networkInterfaces', '${projectName}-vm${i}-networkInterface')
        }
      ]
    }
    osProfile: {
      computerName: '${projectName}-vm${i}'
      adminUsername: adminUsername
      adminPassword: adminPassword
      linuxConfiguration: ((authenticationType == 'password') ? null : linuxConfiguration)
      customData: loadFileAsBase64('jenkins.sh')
    }
  }
  dependsOn: [
    project_vm_1_networkInterface
  ]
}]


resource vNetName_vNetSubnetName 'Microsoft.Network/virtualNetworks/subnets@2021-08-01' = {
  parent: vNet
  name: vNetSubnetName
  properties: {
    addressPrefix: vNetSubnetAddressPrefix
  }
}

resource lb 'Microsoft.Network/loadBalancers@2021-08-01' = {
  name: lbName
  location: location
  sku: {
    name: lbSkuName
  }
  properties: {
    frontendIPConfigurations: [
      {
        name: lbFrontEndName
        properties: {
          publicIPAddress: {
            id: lbPublicIPAddress.id
          }
        }
      }
      {
        name: lbFrontEndNameOutbound
        properties: {
          publicIPAddress: {
            id: lbPublicIPAddressOutbound.id
          }
        }
      }
    ]
    backendAddressPools: [
      {
        name: lbBackendPoolName
      }
      {
        name: lbBackendPoolNameOutbound
      }
    ]
    loadBalancingRules: [
      {
        name: 'myHTTPRule'
        properties: {
          frontendIPConfiguration: {
            id: resourceId('Microsoft.Network/loadBalancers/frontendIPConfigurations', lbName, lbFrontEndName)
          }
          backendAddressPool: {
            id: resourceId('Microsoft.Network/loadBalancers/backendAddressPools', lbName, lbBackendPoolName)
          }
          frontendPort: 8080
          backendPort: 8080
          enableFloatingIP: false
          idleTimeoutInMinutes: 15
          protocol: 'Tcp'
          enableTcpReset: true
          loadDistribution: 'Default'
          disableOutboundSnat: true
          probe: {
            id: resourceId('Microsoft.Network/loadBalancers/probes', lbName, lbProbeName)
          }
        }
      }
    ]
    probes: [
      {
        name: lbProbeName
        properties: {
          protocol: 'Tcp'
          port: 8080
          intervalInSeconds: 5
          numberOfProbes: 2
        }
      }
    ]
    outboundRules: [
      {
        name: 'myOutboundRule'
        properties: {
          allocatedOutboundPorts: 10000
          protocol: 'All'
          enableTcpReset: false
          idleTimeoutInMinutes: 15
          backendAddressPool: {
            id: resourceId('Microsoft.Network/loadBalancers/backendAddressPools', lbName, lbBackendPoolNameOutbound)
          }
          frontendIPConfigurations: [
            {
              id: resourceId('Microsoft.Network/loadBalancers/frontendIPConfigurations', lbName, lbFrontEndNameOutbound)
            }
          ]
        }
      }
    ]
  }
}

resource lbPublicIPAddress 'Microsoft.Network/publicIPAddresses@2021-08-01' = {
  name: lbPublicIpAddressName
  location: location
  sku: {
    name: lbSkuName
  }
  properties: {
    publicIPAddressVersion: 'IPv4'
    publicIPAllocationMethod: 'Static'
  }
}

resource lbPublicIPAddressOutbound 'Microsoft.Network/publicIPAddresses@2021-08-01' = {
  name: lbPublicIPAddressNameOutbound
  location: location
  sku: {
    name: lbSkuName
  }
  properties: {
    publicIPAddressVersion: 'IPv4'
    publicIPAllocationMethod: 'Static'
  }
}

resource nsg 'Microsoft.Network/networkSecurityGroups@2021-08-01' = {
  name: nsgName
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowHTTPInbound'
        properties: {
          protocol: '*'
          sourcePortRange: '*'
          destinationPortRange: '8080'
          sourceAddressPrefix: 'Internet'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 100
          direction: 'Inbound'
        }
      }
    ]
  }
}

resource vNet 'Microsoft.Network/virtualNetworks@2021-08-01' = {
  name: vNetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        vNetAddressPrefix
      ]
    }
  }
}


```
**




## Script de Instalação do Jenkins que será executado pelo Jenkins node A e Jenkins Node B
Crie os arquivos chamado jenkins.sh e ensira os códigos abaixo

```sh

#!/bin/bash
#installing  jenkins
sudo apt install unzip -y
sudo apt install openjdk-11-jre -y
sudo wget -qO - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update && sudo apt-get install jenkins -y
sudo service jenkins restart

sudo mkdir /var/lib/jenkins/jobs

###INSERIR NAS LINHAS ABAIXO AS INFORMACOES DISPONIBILIZADAS PELO SEU PROVEDOR DE NUVEM PARA MAPEAR O FILESERVER em var/lib/jenkins/job 

if [ ! -d "/etc/smbcredentials" ]; then
sudo mkdir /etc/smbcredentials
fi
if [ ! -f "/etc/smbcredentials/NOMESTORAGEACCOUNT.cred" ]; then
    sudo bash -c 'echo "username=##########>" >> /etc/smbcredentials/NOMESTORAGEACCOUNT.cred'
    sudo bash -c 'echo "password=#######################==" >> /etc/smbcredentials/NOMESTORAGEACCOUNT.cred'
fi
sudo chmod 600 /etc/smbcredentials/NOMESTORAGEACCOUNT.cred
sudo bash -c 'echo "//NOMESTORAGEACCOUNT.file.core.windows.net/jenkinsnfs /var/lib/jenkins/jobs cifs nofail,credentials=/etc/smbcredentials/NOMESTORAGEACCOUNT.cred,dir_mode=0777,file_mode=0777,serverino,nosharesock,actimeo=30" >> /etc/fstab'
sudo mount -t cifs //NOMESTORAGEACCOUNT.file.core.windows.net/jenkinsnfs /var/lib/jenkins/jobs -o credentials=/etc/smbcredentials/NOMESTORAGEACCOUNT.cred,dir_mode=0777,file_mode=0777,serverino,nosharesock,actimeo=30


##############FIM  


sudo systemctl stop jenkins
sudo chown -R jenkins:jenkins /var/lib/jenkins/jobs
sudo systemctl start jenkins

```

## MONITORAMENTO NODE A (Grafana and Prometheus)
Crie os arquivos chamado monitoringnodea.bicep e ensira os códigos abaixo

```bicep

// jenkinsnodea.bicep 	
@description('The name of you Virtual Machine.')
param vmName string = 'monitoringnodea'

@description('Username for the Virtual Machine.')
param adminUsername string

@description('Type of authentication to use on the Virtual Machine. SSH key is recommended.')
@allowed([
  'sshPublicKey'
  'password'
])
param authenticationType string = 'password'

param environment string


@description('SSH Key or password for the Virtual Machine. SSH key is recommended.')
//@secure()
param adminPasswordOrKey string

@description('Unique DNS Name for the Public IP used to access the Virtual Machine.')
param dnsLabelPrefix string = toLower('${vmName}-${environment}-${uniqueString(resourceGroup().id)}')

@description('The Ubuntu version for the VM. This will pick a fully patched image of this given Ubuntu version.')
@allowed([
  'Ubuntu-1804'
  'Ubuntu-2004'
  'Ubuntu-2204'
])
param ubuntuOSVersion string = 'Ubuntu-2004'

@description('Location for all resources.')
param location string = resourceGroup().location

@description('The size of the VM')
param vmSize string = 'Standard_B1ms'

@description('Name of the VNET')
param virtualNetworkName string = toLower('${vmName}-${environment}-vnet')

@description('Name of the subnet in the virtual network')
param subnetName string = toLower('${vmName}-${environment}-subnet')

@description('Name of the Network Security Group')
param networkSecurityGroupName string = toLower('${vmName}-${environment}-nsg')



var imageReference = {
  'Ubuntu-1804': {
    publisher: 'Canonical'
    offer: 'UbuntuServer'
    sku: '18_04-lts-gen2'
    version: 'latest'
  }
  'Ubuntu-2004': {
    publisher: 'Canonical'
    offer: '0001-com-ubuntu-server-focal'
    sku: '20_04-lts-gen2'
    version: 'latest'
  }
  'Ubuntu-2204': {
    publisher: 'Canonical'
    offer: '0001-com-ubuntu-server-jammy'
    sku: '22_04-lts-gen2'
    version: 'latest'
  }
}

var publicIPAddressName = toLower('${vmName}-${environment}-PublicIp')
var networkInterfaceName = toLower('${vmName}-${environment}-nic')
var osDiskType = 'Standard_LRS'
var subnetAddressPrefix = '10.1.0.0/24'
var addressPrefix = '10.1.0.0/16'

var linuxConfiguration = {
  disablePasswordAuthentication: false
  ssh: {
    publicKeys: [
      {
        path: '/home/${adminUsername}/.ssh/authorized_keys'
        keyData: adminPasswordOrKey
      }
    ]
  }
}


resource networkInterface 'Microsoft.Network/networkInterfaces@2021-05-01' = {
  name: networkInterfaceName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: toLower('${vmName}-${environment}-ipconfig')
        properties: {
          subnet: {
            id: subnet.id
          }
          privateIPAllocationMethod: 'Dynamic'
          publicIPAddress: {
            id: publicIPAddress.id
          }
        }
      }
    ]
    networkSecurityGroup: {
      id: networkSecurityGroup.id
    }
  }
}

resource networkSecurityGroup 'Microsoft.Network/networkSecurityGroups@2021-05-01' = {
  name: networkSecurityGroupName
  location: location
  properties: {
    securityRules: [
      {
        name: 'SSH'
        properties: {
          priority: 1000
          protocol: 'Tcp'
          access: 'Allow'
          direction: 'Inbound'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '22'
        }
      }

      {
        name: 'Prometheus'
        properties: {
          priority: 1001
          protocol: 'Tcp'
          access: 'Allow'
          direction: 'Inbound'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '9090'
        }
      }

      {
        name: 'Grafana'
        properties: {
          priority: 1002
          protocol: 'Tcp'
          access: 'Allow'
          direction: 'Inbound'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '3000'
        }
      }

    ]
  }
}

resource virtualNetwork 'Microsoft.Network/virtualNetworks@2021-05-01' = {
  name: virtualNetworkName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        addressPrefix
      ]
    }
  }
}

resource subnet 'Microsoft.Network/virtualNetworks/subnets@2021-05-01' = {
  parent: virtualNetwork
  name: subnetName
  properties: {
    addressPrefix: subnetAddressPrefix
    privateEndpointNetworkPolicies: 'Enabled'
    privateLinkServiceNetworkPolicies: 'Enabled'
  }
}

resource publicIPAddress 'Microsoft.Network/publicIPAddresses@2021-05-01' = {
  name: publicIPAddressName
  location: location
  sku: {
    name: 'Basic'
  }
  properties: {
    publicIPAllocationMethod: 'Dynamic'
    publicIPAddressVersion: 'IPv4'
    dnsSettings: {
      domainNameLabel: dnsLabelPrefix
    }
    idleTimeoutInMinutes: 4
  }
}

resource vm 'Microsoft.Compute/virtualMachines@2021-11-01' = {
  name: toLower('${vmName}-${environment}') 
  location: location
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    storageProfile: {
      osDisk: {
        
        createOption: 'FromImage'
        managedDisk: {
          storageAccountType: osDiskType
        }
      }
      imageReference: imageReference[ubuntuOSVersion]
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: networkInterface.id
        }
      ]
    }
    
    osProfile: {
      computerName: vmName
      adminUsername: adminUsername
      adminPassword: adminPasswordOrKey
      linuxConfiguration: ((authenticationType == 'password') ? null : linuxConfiguration)
      customData: loadFileAsBase64('./monitoring.sh')
    }


  }
}


output adminUsername string = adminUsername
output hostname string = publicIPAddress.properties.dnsSettings.fqdn
output sshCommand string = 'ssh ${adminUsername}@${publicIPAddress.properties.dnsSettings.fqdn}'


```

## MONITORAMENTO NODE B (Grafana and Prometheus)
Crie os arquivos chamado monitoringnodeb.bicep e ensira os códigos abaixo

```bicep
// monitoringnodeb.bicep 	
@description('The name of you Virtual Machine.')
param vmName string = 'monitoringnodea'

@description('Username for the Virtual Machine.')
param adminUsername string

@description('Type of authentication to use on the Virtual Machine. SSH key is recommended.')
@allowed([
  'sshPublicKey'
  'password'
])
param authenticationType string = 'password'

param environment string


@description('SSH Key or password for the Virtual Machine. SSH key is recommended.')
//@secure()
param adminPasswordOrKey string

@description('Unique DNS Name for the Public IP used to access the Virtual Machine.')
param dnsLabelPrefix string = toLower('${vmName}-${environment}-${uniqueString(resourceGroup().id)}')

@description('The Ubuntu version for the VM. This will pick a fully patched image of this given Ubuntu version.')
@allowed([
  'Ubuntu-1804'
  'Ubuntu-2004'
  'Ubuntu-2204'
])
param ubuntuOSVersion string = 'Ubuntu-2004'

@description('Location for all resources.')
param location string = resourceGroup().location

@description('The size of the VM')
param vmSize string = 'Standard_B1ms'

@description('Name of the VNET')
param virtualNetworkName string = toLower('${vmName}-${environment}-vnet')

@description('Name of the subnet in the virtual network')
param subnetName string = toLower('${vmName}-${environment}-subnet')

@description('Name of the Network Security Group')
param networkSecurityGroupName string = toLower('${vmName}-${environment}-nsg')


var imageReference = {
  'Ubuntu-1804': {
    publisher: 'Canonical'
    offer: 'UbuntuServer'
    sku: '18_04-lts-gen2'
    version: 'latest'
  }
  'Ubuntu-2004': {
    publisher: 'Canonical'
    offer: '0001-com-ubuntu-server-focal'
    sku: '20_04-lts-gen2'
    version: 'latest'
  }
  'Ubuntu-2204': {
    publisher: 'Canonical'
    offer: '0001-com-ubuntu-server-jammy'
    sku: '22_04-lts-gen2'
    version: 'latest'
  }
}

var publicIPAddressName = toLower('${vmName}-${environment}-PublicIp')
var networkInterfaceName = toLower('${vmName}-${environment}-nic')
var osDiskType = 'Standard_LRS'
var subnetAddressPrefix = '10.2.0.0/24'
var addressPrefix = '10.2.0.0/16'

var linuxConfiguration = {
  disablePasswordAuthentication: false
  ssh: {
    publicKeys: [
      {
        path: '/home/${adminUsername}/.ssh/authorized_keys'
        keyData: adminPasswordOrKey
      }
    ]
  }
}



resource networkInterface 'Microsoft.Network/networkInterfaces@2021-05-01' = {
  name: networkInterfaceName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: toLower('${vmName}-${environment}-ipconfig')
        properties: {
          subnet: {
            id: subnet.id
          }
          privateIPAllocationMethod: 'Dynamic'
          publicIPAddress: {
            id: publicIPAddress.id
          }
        }
      }
    ]
    networkSecurityGroup: {
      id: networkSecurityGroup.id
    }
  }
}

resource networkSecurityGroup 'Microsoft.Network/networkSecurityGroups@2021-05-01' = {
  name: networkSecurityGroupName
  location: location
  properties: {
    securityRules: [
      {
        name: 'SSH'
        properties: {
          priority: 1000
          protocol: 'Tcp'
          access: 'Allow'
          direction: 'Inbound'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '22'
        }
      }

      {
        name: 'Prometheus'
        properties: {
          priority: 1001
          protocol: 'Tcp'
          access: 'Allow'
          direction: 'Inbound'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '9090'
        }
      }

      {
        name: 'Grafana'
        properties: {
          priority: 1002
          protocol: 'Tcp'
          access: 'Allow'
          direction: 'Inbound'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '3000'
        }
      }

    ]
  }
}

resource virtualNetwork 'Microsoft.Network/virtualNetworks@2021-05-01' = {
  name: virtualNetworkName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        addressPrefix
      ]
    }
  }
}

resource subnet 'Microsoft.Network/virtualNetworks/subnets@2021-05-01' = {
  parent: virtualNetwork
  name: subnetName
  properties: {
    addressPrefix: subnetAddressPrefix
    privateEndpointNetworkPolicies: 'Enabled'
    privateLinkServiceNetworkPolicies: 'Enabled'
  }
}

resource publicIPAddress 'Microsoft.Network/publicIPAddresses@2021-05-01' = {
  name: publicIPAddressName
  location: location
  sku: {
    name: 'Basic'
  }
  properties: {
    publicIPAllocationMethod: 'Dynamic'
    publicIPAddressVersion: 'IPv4'
    dnsSettings: {
      domainNameLabel: dnsLabelPrefix
    }
    idleTimeoutInMinutes: 4
  }
}

resource vm 'Microsoft.Compute/virtualMachines@2021-11-01' = {
  name: toLower('${vmName}-${environment}') 
  location: location
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    storageProfile: {
      osDisk: {
        
        createOption: 'FromImage'
        managedDisk: {
          storageAccountType: osDiskType
        }
      }
      imageReference: imageReference[ubuntuOSVersion]
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: networkInterface.id
        }
      ]
    }
    
    osProfile: {
      computerName: vmName
      adminUsername: adminUsername
      adminPassword: adminPasswordOrKey
      linuxConfiguration: ((authenticationType == 'password') ? null : linuxConfiguration)
      customData: loadFileAsBase64('./monitoringnodeb.sh')
    }

  }
}

output adminUsername string = adminUsername
output hostname string = publicIPAddress.properties.dnsSettings.fqdn
output sshCommand string = 'ssh ${adminUsername}@${publicIPAddress.properties.dnsSettings.fqdn}'

```

***



## Script de Instalação do grafana e prometheus que será executado pelo servidor MONITORAMENTO NODE A (Grafana and Prometheus)
Crie os arquivos chamado monitoringnodea.sh e ensira os códigos abaixo


```bicep
#!/bin/bash

sudo snap install docker 
sudo sleep 200s

cat <<EOF >> /home/victor/prometheus.yml
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'docker'
         # metrics_path defaults to '/metrics'
         # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9323']

  - job_name: 'jenkins'
    metrics_path: /prometheus
    static_configs:
      - targets: ['10.1.0.4:8080']
EOF
sudo docker run --name prometheus -d -p 9090:9090 -v /home/victor/prometheus.yml:/etc/prometheus/prometheus.yml  prom/prometheus
sudo docker run -d --name grafana -p 3000:3000 grafana/grafana

sudo apt install unzip -y

```



## Script de Instalação do grafana e prometheus que será executado pelo servidor MONITORAMENTO NODE B (Grafana and Prometheus)
Crie os arquivos chamado monitoringnodeb.sh e ensira os códigos abaixo


```sh
#!/bin/bash

sudo snap install docker 
sudo sleep 200s

cat <<EOF >> /home/victor/prometheus.yml
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'docker'
         # metrics_path defaults to '/metrics'
         # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9323']
  - job_name: 'jenkins'
    metrics_path: /prometheus
    static_configs:
      - targets: ['10.2.0.4:8080']
EOF
sudo docker run --name prometheus -d -p 9090:9090 -v /home/victor/prometheus.yml:/etc/prometheus/prometheus.yml  prom/prometheus
sudo docker run -d --name grafana -p 3000:3000 grafana/grafana
sudo apt install unzip -y


```


# Criacao da Pipeline de AUtomacao do ambiente 

Apos criar todos os arquivos corretamente, voce tera 6 arquivos no total
![image](https://user-images.githubusercontent.com/28166733/227672352-046db9a6-6c7e-4bd2-9ab0-b65d02c96733.png)

## Criacao da pipeline de execucao do codigo 
Clique em Pipelines e create pipeline

<img width="656" alt="image" src="https://user-images.githubusercontent.com/28166733/221449168-a9515890-a90b-4827-a0ac-e770d91511f2.png">

Apos clicar em create pipeline seleciona a opcao "Azure Repo Git"
<img width="596" alt="image" src="https://user-images.githubusercontent.com/28166733/221449217-06b44019-7e00-406a-bd7e-fa44a1e54981.png">

Selecione o repositorio MBA 

<img width="476" alt="image" src="https://user-images.githubusercontent.com/28166733/221449265-483df6db-bdef-4be3-80d2-a55c8d2eb332.png">

Clique em Starter Pipeline 

<img width="349" alt="image" src="https://user-images.githubusercontent.com/28166733/221449313-46e15993-ebe0-49b7-837c-7716c7d89e98.png">

Cole a pipeline e clique em save and run 

```yml
  pool:
   vmImage: "windows-latest"

 trigger: none

 stages:
  - stage: dev
    displayName: "DEV"    
    jobs:
      - job: azureResourcesBicep
        displayName: "Deploy IAC"
        steps:
            

        - task: AzureResourceManagerTemplateDeployment@3
          inputs:
            deploymentScope: 'Resource Group'
            azureResourceManagerConnection: 'MyPipelineDevOps'
            subscriptionId:  <SEU-ID-DE-SUBSCRICAO>
            action: 'Create Or Update Resource Group'
            resourceGroupName: 'rg-dev-jenkins'
            location: 'Brazil South'
            templateLocation: 'Linked artifact'
            csmFile: 'arm/jenkinsnodes.bicep'
            overrideParameters: '-projectName dev-jenkins -adminUsername USERNAME -adminPassword SENHA#*'
            deploymentMode: 'Incremental'
            deploymentName: 'DeployPipelineTemplate'


        - task: AzureResourceManagerTemplateDeployment@3
          inputs:
            deploymentScope: 'Resource Group'
            azureResourceManagerConnection: 'MyPipelineDevOps'
            subscriptionId: <SEU-ID-DE-SUBSCRICAO>
            action: 'Create Or Update Resource Group'
            resourceGroupName: 'rg-dev-jenkins'
            location: 'Brazil South'
            templateLocation: 'Linked artifact'
            csmFile: 'arm/monitoringnodea.bicep'
            overrideParameters: '-adminUsername USER -environment dev -adminPasswordOrKey Senha#*'
            deploymentMode: 'Incremental'
            deploymentName: 'DeployPipelineTemplate'

        - task: AzureResourceManagerTemplateDeployment@3
          inputs:
            deploymentScope: 'Resource Group'
            azureResourceManagerConnection: 'MyPipelineDevOps'
            subscriptionId: <SEU-ID-DE-SUBSCRICAO>
            action: 'Create Or Update Resource Group'
            resourceGroupName: 'rg-dev-jenkins'
            location: 'Brazil South'
            templateLocation: 'Linked artifact'
            csmFile: 'arm/monitoringnodeb.bicep'
            overrideParameters: '-adminUsername USER -environment dev -adminPasswordOrKey Senha#*'
            deploymentMode: 'Incremental'
            deploymentName: 'DeployPipelineTemplate'
        



        - task: AzurePowerShell@5
          inputs:
            azureSubscription: 'MyPipelineDevOps'
            ScriptType: 'InlineScript'
            Inline: |
              $virtualNetwork1 = Get-AzVirtualNetwork `
                -ResourceGroupName rg-dev-jenkins `
                -Name dev-jenkins-vnet `

              Add-AzVirtualNetworkPeering `
                -Name "monitoringnodea-dev-vnet-dev-jenkins-vnet" `
                -VirtualNetwork $virtualNetwork1 `
                -RemoteVirtualNetworkId /subscriptions/<SEU-ID-DE-SUBSCRICAO>/resourceGroups/rg-dev-jenkins/providers/Microsoft.Network/virtualNetworks/monitoringnodea-dev-vnet

              $virtualNetwork2 = Get-AzVirtualNetwork `
                -ResourceGroupName rg-dev-jenkins `
                -Name monitoringnodea-dev-vnet `
                            

              Add-AzVirtualNetworkPeering `
                -Name "monitoringnodea-dev-vnet-dev-jenkins-vnet" `
                -VirtualNetwork $virtualNetwork2 `
                -RemoteVirtualNetworkId /subscriptions/<SEU-ID-DE-SUBSCRICAO>/resourceGroups/rg-dev-jenkins/providers/Microsoft.Network/virtualNetworks/dev-jenkins-vnet  
              

              
              Add-AzVirtualNetworkPeering `
                -Name "monitoringnodeb-dev-vnet-dev-jenkins-vnet" `
                -VirtualNetwork $virtualNetwork1 `
                -RemoteVirtualNetworkId /subscriptions/<SEU-ID-DE-SUBSCRICAO>/resourceGroups/rg-dev-jenkins/providers/Microsoft.Network/virtualNetworks/monitoringnodeb-dev-vnet
              
              $virtualNetwork3 = Get-AzVirtualNetwork `
                -ResourceGroupName rg-dev-jenkins `
                -Name monitoringnodeb-dev-vnet `
              
              
              Add-AzVirtualNetworkPeering `
                -Name "monitoringnodeb-dev-vnet-dev-jenkins-vnet" `
                -VirtualNetwork $virtualNetwork3 `
                -RemoteVirtualNetworkId /subscriptions/<SEU-ID-DE-SUBSCRICAO>/resourceGroups/rg-dev-jenkins/providers/Microsoft.Network/virtualNetworks/dev-jenkins-vnet
            azurePowerShellVersion: 'LatestVersion'



```

Apos clicar em Save and Run, voce recebera uma mensagem para adiciionar permissao oao recurso, clique em view 

<img width="760" alt="image" src="https://user-images.githubusercontent.com/28166733/221452676-86450bd8-b060-4a59-873f-254a4cbfb302.png">

apos, clique em Permit 

<img width="275" alt="image" src="https://user-images.githubusercontent.com/28166733/221452713-ef63048e-5745-4c15-a9b4-4fa8c3a29668.png">



Verifique os passsos apos rodar a pipeline


<img width="797" alt="image" src="https://user-images.githubusercontent.com/28166733/221453532-0e43fe7d-601f-4646-96ff-deb811f00fad.png">


Apos rodar a pipeline abra o servidor jenkins criado atraves do fqdn com a porta 8080, 

<img width="776" alt="image" src="https://user-images.githubusercontent.com/28166733/221480776-38cbb479-f049-43e8-a8e0-8fd14b79d3bf.png">

Relize o comando cat conforme informado na tela inicial do jenkins e instale todos os pacotes 

```ssh
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

apos a instalação, busque nos plugins do Jenkins o plugin do Prometheus e adicione ao jenkins, realize a instalação para o jenkinsnodea e jenkinsnodeb, 
nao esqueca que os mesmos estarao por tras de um load balancer, recomendado que realize a configuracao do primeiro jenkins e desligue-o para que as 
configuracoes iniciais possa ser realizada no nodeb via redirecionamento do trafego do balancer, após configurar o node b desligue-o, rode a pipeline novamentte 
ou religue as VMs novamente via portal para que as mesmas iniciem com as configuracoes iniciais com o plugin instalado corretamente,

![image](https://user-images.githubusercontent.com/28166733/227682480-1708677f-e773-44d0-b381-e9a1bd89a1d2.png)





***


# GRAFANA 

acesse a tela do grafana atraves do link gerado nda maquina monitoringnodea no navegador utilizando a porta 3000 do grafana

apos criar senha nova para o usuario admin va em datasoruces e clique em adicionar datasource e procure prometheus


![image](https://user-images.githubusercontent.com/28166733/221679253-e3343bcb-1242-4d13-8c49-11fa838d8eed.png)

DIgite o ip privado 10.3.0.4 do servidor monitoringnodea onde esta localizado o prometheus no campo conforme imagem abaixo e clique em salver e testar configuracao



![image](https://user-images.githubusercontent.com/28166733/221680012-65c6dc0a-d440-43a7-be27-165ec6f3eb24.png)

Clique em Dashboard e va  em import 

 ![image](https://user-images.githubusercontent.com/28166733/221689791-bfbc0ab8-54f4-474e-8ec7-b0dee4719cbd.png)
 
 digite o codigo 9524 e clique em load para carregar um dashboard personalizado pela jenkins feito para o grafana 
 
![image](https://user-images.githubusercontent.com/28166733/221682003-ab6b0840-4604-45d7-957b-20af526ab5e9.png)

apos clicar em load, clique em import 

![image](https://user-images.githubusercontent.com/28166733/221682334-cee71bcc-0718-4c4b-8ee2-c4f519cae1e2.png)

 


Realizar o mesmo passo no servidor monitoringnodeb e adicionar o IP Local do mesmo 10.4.0.4


acesse a tela do grafana atraves do link gerado nda maquina monitoringnodea no navegador utilizando a porta 3000 do grafana



apos criar senha nova para o usuario admin va em datasoruces e clique em adicionar datasource e procure prometheus


![image](https://user-images.githubusercontent.com/28166733/221679253-e3343bcb-1242-4d13-8c49-11fa838d8eed.png)


DIgite o ip privado 10.4.0.4 do servidor monitoringnodea onde esta localizado o prometheus no campo conforme imagem abaixo e clique em salver e testar configuracao

![image](https://user-images.githubusercontent.com/28166733/221688544-97b89d2e-e205-4f58-85b7-3d22445466e6.png)




Clique em Dashboard e va  em import 

 ![image](https://user-images.githubusercontent.com/28166733/221689791-bfbc0ab8-54f4-474e-8ec7-b0dee4719cbd.png)
 
 digite o codigo 9524 e clique em load para carregar um dashboard personalizado pela jenkins feito para o grafana 
 
![image](https://user-images.githubusercontent.com/28166733/221682003-ab6b0840-4604-45d7-957b-20af526ab5e9.png)

apos clicar em load, clique em import 




![image](https://user-images.githubusercontent.com/28166733/221682334-cee71bcc-0718-4c4b-8ee2-c4f519cae1e2.png)

Após realizar o imporot do código você  receberá está visão do dashboard, execute este processo na vm com o grafana monitoringnodeb:3000

![image](https://user-images.githubusercontent.com/28166733/221945287-a89352c8-57b5-483d-9c81-28e5ffaca750.png)

