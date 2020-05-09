---
title: "Azure Terraform"
date: 2020-05-09T20:09:36+02:00
draft: false
toc: false
images:
tags: 
  - azure
  - terraform
---
Cómo empezar a utilizar Terraform sobre linux para crear infraestructure-as-code en Azure. 
Resumen y notas a partir de https://docs.microsoft.com/es-es/azure/developer/terraform/install-configure

### 1.- Instalar el cliente de azure (https://docs.microsoft.com/es-es/cli/azure/install-azure-cli-apt?view=azure-cli-latest)

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### 2.- Instalar Terraform
Descargar (https://www.terraform.io/downloads.html) descomprimir y meter en el path. Al descomprimir solo hay un ejecutable. Copiar al $HOME/bin

### 3.- Login en la cuenta de azure
```bash
az login
```

Debe devolver información sobre la subscripción
```json
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "isDefault": true,
    "managedByTenants": [],
    "name": "Azure subscription 1",
    "state": "Enabled",
    "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "user": {
      "name": "xx@xxxxx.xxx",
      "type": "user"
    }
  }
]
```

## Configurar terraform
https://docs.microsoft.com/es-es/azure/developer/terraform/install-configure

Hay que coger el subscription id y guardarlo en una variable

```bash
az account list --query "[].{name:name, subscriptionId:id, tenantId:tenantId}"
```

Devolverá el subscriptionId
```json
[
  {
    "name": "Azure subscription 1",
    "subscriptionId": "0077e616-271a-4d26-8bf7-28dedbf0b87c",
    "tenantId": "c1f19454-3a2e-4bfa-966e-4e4e72ff7803"
  }
]
```

Hay que seleccionar la subscripción con la que queremos trabajar

```bash
export SUBSCRIPTION_ID="0077e616-271a-4d26-8bf7-28dedbf0b87c"
az account set --subscription="${SUBSCRIPTION_ID}"
```

y crear crear un principal para automatizar de forma segura (recomiendan no usar la cuenta de usuario normal)

```bash
az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/${SUBSCRIPTION_ID}"
```


```json
{
  "appId": "xxxxxxxa-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "displayName": "azure-cli-YYYY-MM-DD-hh-mm-ss",
  "name": "http://azure-cli-YYYY-MM-DD-hh-mm-ss",
  "password": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```
HAY QUE GUARDAR ESTOS DATOS!!!!!
Para ello creamos un script para inicializar variables de entorno (azure.env) en futuras sesiones

```bash
#!/bin/sh
echo "Setting environment variables for Terraform"
export ARM_SUBSCRIPTION_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export ARM_CLIENT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"    #your_appId
export ARM_CLIENT_SECRET="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"   #your_password
export ARM_TENANT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"   #your_tenant_id

# Not needed for public, required for usgovernment, german, china
#export ARM_ENVIRONMENT=public
```

Para usar la cuenta de usuario ver: https://www.terraform.io/docs/providers/azurerm/guides/azure_cli.html


Definimos en nuestro entorno las variables anteriores.
```bash
source azure.env
```

Ya podriamos ejecutar scripts .tf pero primero hay que inicializar el entorno de terraform (init - descarga los plugins para azure)

```bash
terraform init
terraform plan
terraform apply -auto-approve
```


### 4.- Crear una maquina virtual 
Páginas relacionadas:

  - https://docs.microsoft.com/es-es/azure/developer/terraform/create-linux-virtual-machine-with-infrastructure)
  - https://www.terraform.io/docs/providers/azurerm/
  - https://www.terraform.io/docs/providers/azurerm/r/subnet.html

Creamos una pequeña maquina virtual Standard_B1s mediante el fichero test.tf.
Nótese el uso de variables con valores por defecto que nos dan flexibilidad y no tener que repetir valores.
```terraform
provider "azurerm" {
  # The "feature" block is required for AzureRM provider 2.x. 
  # If you are using version 1.x, the "features" block is not allowed.
  version = "~>2.0"
  features {}
}

variable "rg" {
  type = string
  default = "rgtf"  
}

variable "location" {
  type = string
  default = "westeurope"
}


variable "envtag" {
  type = string
  default = "Terraform Demo"
}



# Create a resource group if it doesn't exist
resource "azurerm_resource_group" "myterraformgroup" {
    name     = var.rg
    location = var.location

    tags = {
        environment = var.envtag
    }
}

# Create virtual network
resource "azurerm_virtual_network" "myterraformnetwork" {
    name                = "myVnet"
    address_space       = ["10.0.0.0/16"]
    location            = var.location
    resource_group_name = azurerm_resource_group.myterraformgroup.name

    tags = {
        environment = var.envtag
    }
}

# Create subnet
resource "azurerm_subnet" "myterraformsubnet" {
    name                 = "mySubnet"
    resource_group_name  = azurerm_resource_group.myterraformgroup.name
    virtual_network_name = azurerm_virtual_network.myterraformnetwork.name
    address_prefixes     = ["10.0.1.0/24"]
}

# Create public IPs
resource "azurerm_public_ip" "myterraformpublicip" {
    name                         = "myPublicIP"
    location                     = var.location
    resource_group_name          = azurerm_resource_group.myterraformgroup.name
    allocation_method            = "Dynamic"

    tags = {
        environment = var.envtag
    }
}

# Create Network Security Group and rule
resource "azurerm_network_security_group" "myterraformnsg" {
    name                = "myNetworkSecurityGroup"
    location            = var.location
    resource_group_name = azurerm_resource_group.myterraformgroup.name
    
    security_rule {
        name                       = "SSH"
        priority                   = 1001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
    }

    tags = {
        environment = var.envtag
    }
}

# Create network interface
resource "azurerm_network_interface" "myterraformnic" {
    name                      = "myNIC"
    location                  = var.location
    resource_group_name       = azurerm_resource_group.myterraformgroup.name

    ip_configuration {
        name                          = "myNicConfiguration"
        subnet_id                     = azurerm_subnet.myterraformsubnet.id
        private_ip_address_allocation = "Dynamic"
        public_ip_address_id          = azurerm_public_ip.myterraformpublicip.id
    }

    tags = {
        environment = var.envtag
    }
}

# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
    network_interface_id      = azurerm_network_interface.myterraformnic.id
    network_security_group_id = azurerm_network_security_group.myterraformnsg.id
}

# Generate random text for a unique storage account name
resource "random_id" "randomId" {
    keepers = {
        # Generate a new ID only when a new resource group is defined
        resource_group = azurerm_resource_group.myterraformgroup.name
    }
    
    byte_length = 8
}

# Create storage account for boot diagnostics
resource "azurerm_storage_account" "mystorageaccount" {
    name                        = "diag${random_id.randomId.hex}"
    resource_group_name         = azurerm_resource_group.myterraformgroup.name
    location                    = var.location
    account_tier                = "Standard"
    account_replication_type    = "LRS"

    tags = {
        environment = "Terraform Demo"
    }
}

# Create virtual machine
resource "azurerm_linux_virtual_machine" "myterraformvm" {
    name                  = "myvm"
    location              = var.location
    resource_group_name   = azurerm_resource_group.myterraformgroup.name
    network_interface_ids = [azurerm_network_interface.myterraformnic.id]
    size                  = "Standard_B1s"

    os_disk {
        name              = "myOsDisk"
        caching           = "ReadWrite"
        storage_account_type = "Standard_LRS"
    }

    source_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "18.04-LTS"
        version   = "latest"
    }

    computer_name  = "myvm"
    admin_username = "azureuser"
    disable_password_authentication = true
        
    admin_ssh_key {
        username       = "azureuser"
        public_key     = file("/home/educhana/.ssh/id_rsa.pub")
    }

    boot_diagnostics {
        storage_account_uri = azurerm_storage_account.mystorageaccount.primary_blob_endpoint
    }

    tags = {
        environment = var.envtag
    }
}


```


hacemos 

```bash
terraform apply -auto-approve
```

Miramos qué ips tienes las máquinas de nuestro resorce group para conectarnos a ellas

```bash
az vm show --resource-group rgtf --name myvm -d --query [publicIps] --output tsv

ssh azureuser@publicIp
```


Para destruir los recursos y vms que hemos creado:

```bash
terraform plan -destroy
terraform destroy
```



### Observaciones

##### Lista de imagenes para maquinas
```bash
az vm image list --output table
```



#### Importar recurso existente a terraform

Cuidado! Terraform pasa a manejar los recursos y los destruirá cuando hagamos un destroy

```bash
terraform import azurerm_resource_group.myterraformgroup /subscriptions/${ARM_SUBSCRIPTION_ID}/resourceGroups/rgtf
```


#### Utilizar recursos ya existentes sin que Terraform los maneje
Si no quieres que terraform maneje algun recurso, no hay que incluirlo, sino referenciarlo por su nombre

En vez de:

```
    resource_group_name = azurerm_resource_group.myterraformgroup.name
```

usar:

```
    resource_group_name = "rg1"
```

