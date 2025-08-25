# Azure-Virtual-Network-Peering Guide:

Azure CLI commands and terraform configuration

 # Objective:
To create a secure VNet peering connection between two different VNets

# Step-by-Step Setup Using Azure CLI:

1️⃣ Create Resource Group
az group create --name MyResourceGroup --location eastus

2️⃣ Create Two VNets:

az network vnet create \
  --resource-group MyResourceGroup \
  --name VNetA \
  --address-prefix 10.0.0.0/16 \
  --subnet-name SubnetA \
  --subnet-prefix 10.0.1.0/24

az network vnet create \
  --resource-group MyResourceGroup \
  --name VNetB \
  --address-prefix 10.1.0.0/16 \
  --subnet-name SubnetB \
  --subnet-prefix 10.1.1.0/24

# 3️⃣ Create VNet Peerings


# 4️⃣ Deploy Virtual Machines:
az vm create \
  --resource-group MyResourceGroup \
  --name VM1 \
  --image UbuntuLTS \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name VNetA \
  --subnet SubnetA

az vm create \
  --resource-group MyResourceGroup \
  --name VM2 \
  --image UbuntuLTS \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name VNetB \
  --subnet SubnetB

##
🚀 Test Connectivity with given bash cmd 

SSH into VM1:
az vm ssh --name VM1 --resource-group MyResourceGroup


Ping VM2’s private IP:
ping 10.1.1.4

# SSH from VM1 to VM2 with security rules access:

ssh azureuser@10.1.1.4

# Cleanup
# bash: -az group delete --name MyResourceGroup --yes --no-wait


## Terraform setup with Network Security Groups (NSGs) and rules:

Port SSH (22) and ICMP (ping) are allowed between the two VMNets.
Outbound internet remains allowed by default.
Each subnet gets an NSG applied.


# 🛠️ Terraform Setup with NSG Rules:
main.tf
provider "azurerm" {
  features {}
}

# Resource Group
resource "azurerm_resource_group" "rg" {
  name     = "MyResourceGroup"
  location = "East US"
}

# VNet A
resource "azurerm_virtual_network" "vnet_a" {
  name                = "VNetA"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_subnet" "subnet_a" {
  name                 = "SubnetA"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet_a.name
  address_prefixes     = ["10.0.1.0/24"]
}

# VNet B
resource "azurerm_virtual_network" "vnet_b" {
  name                = "VNetB"
  address_space       = ["10.1.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_subnet" "subnet_b" {
  name                 = "SubnetB"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet_b.name
  address_prefixes     = ["10.1.1.0/24"]
}

# NSG for VNet A
resource "azurerm_network_security_group" "nsg_a" {
  name                = "NSG-A"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "Allow-SSH-From-VNetB"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "10.1.0.0/16"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow-ICMP-From-VNetB"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Icmp"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "10.1.0.0/16"
    destination_address_prefix = "*"
  }
}

# NSG for VNet B
resource "azurerm_network_security_group" "nsg_b" {
  name                = "NSG-B"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "Allow-SSH-From-VNetA"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "10.0.0.0/16"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow-ICMP-From-VNetA"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Icmp"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "10.0.0.0/16"
    destination_address_prefix = "*"
  }
}


