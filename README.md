# Azure-Virtual-Network-Peering Guide:

Azure CLI commands (az):

 🎯 Objective

To create a secure VNet peering connection between two different VNets in the same Azure region and route traffic privately between their virtual machines.
🧩 Use Case

VNet-A: Application server
VNet-B: Database server
Goal: Allow the app server in VNet-A to access the database in VNet-B over private IPs.

🏗️ Step-by-Step Setup Using Azure CLI:
1️⃣ Create Resource Group
az group create --name MyResourceGroup --location eastus

2️⃣ Create Two VNets
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

3️⃣ Create VNet Peerings
# Peering from VNetA to VNetB
az network vnet peering create \
  --name VNetA-to-VNetB \
  --resource-group MyResourceGroup \
  --vnet-name VNetA \
  --remote-vnet VNetB \
  --allow-vnet-access

# Peering from VNetB to VNetA
az network vnet peering create \
  --name VNetB-to-VNetA \
  --resource-group MyResourceGroup \
  --vnet-name VNetB \
  --remote-vnet VNetA \
  --allow-vnet-access

4️⃣ Deploy Virtual Machines
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

🚀 Test Connectivity

SSH into VM1:

az vm ssh --name VM1 --resource-group MyResourceGroup


Ping VM2’s private IP:
ping 10.1.1.4


SSH from VM1 to VM2 (if security rules allow):

ssh azureuser@10.1.1.4

🧼 Cleanup
az group delete --name MyResourceGroup --yes --no-wait


# Terraform setup with Network Security Groups (NSGs) and rules #

SSH (22) and ICMP (ping) are allowed between the two VNets.

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

# Associate NSG with Subnets
resource "azurerm_subnet_network_security_group_association" "subnet_a_nsg" {
  subnet_id                 = azurerm_subnet.subnet_a.id
  network_security_group_id = azurerm_network_security_group.nsg_a.id
}

resource "azurerm_subnet_network_security_group_association" "subnet_b_nsg" {
  subnet_id                 = azurerm_subnet.subnet_b.id
  network_security_group_id = azurerm_network_security_group.nsg_b.id
}

# VNet Peering
resource "azurerm_virtual_network_peering" "peer_a_to_b" {
  name                      = "VNetA-to-VNetB"
  resource_group_name       = azurerm_resource_group.rg.name
  virtual_network_name      = azurerm_virtual_network.vnet_a.name
  remote_virtual_network_id = azurerm_virtual_network.vnet_b.id
  allow_virtual_network_access = true
}

resource "azurerm_virtual_network_peering" "peer_b_to_a" {
  name                      = "VNetB-to-VNetA"
  resource_group_name       = azurerm_resource_group.rg.name
  virtual_network_name      = azurerm_virtual_network.vnet_b.name
  remote_virtual_network_id = azurerm_virtual_network.vnet_a.id
  allow_virtual_network_access = true
}

# outputs.tf
output "vnet_a_id" {
  value = azurerm_virtual_network.vnet_a.id
}

output "vnet_b_id" {
  value = azurerm_virtual_network.vnet_b.id
}
