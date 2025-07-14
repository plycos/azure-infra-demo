# (Assumed) Resource group
az group create \
--name LabRG \
--location centralus

# VNet + subnets
az network vnet create \
--resource-group LabRG \
--name LabVNet \
--address-prefixes 10.1.0.0/16 \
--subnet-name appSubnet \
--subnet-prefixes 10.1.1.0/24 \
--location centralus

az network vnet subnet create \
--resource-group LabRG \
--vnet-name LabVNet \
--name dbSubnet \
--address-prefixes 10.1.2.0/24

# Private DNS zone & linking
az network private-dns zone create \
--resource-group LabRG \
--name privatelink.database.windows.net

az network private-dns link vnet create \
--resource-group LabRG \
--zone-name privatelink.database.windows.net \
--name link-LabVNet \
--virtual-network LabVNet \
--registration-enabled false

az network private-endpoint dns-zone-group create \
--resource-group LabRG \
--endpoint-name sql-pe \
--name sql-zone-group \
--private-dns-zone privatelink.database.windows.net \
--zone-name privatelink.database.windows.net


# 1. Point at your AAD admin account
export MSEntraAccount="pwlycos_hotmail.com#EXT#@pwlycoshotmail.onmicrosoft.com"
export MSEntraSID=$(az ad user show \
--id $MSEntraAccount \
--query objectId -o tsv)

# 2. Create the logical SQL Server with AAD-only (passwordless) auth
az sql server create \
--resource-group LabRG \
--name labrg-sqlserver \
--location centralus \
--enable-ad-only-auth \
--external-admin-principal-type User \
--external-admin-name $MSEntraAccount \
--external-admin-sid $MSEntraSID

# 3. Create the GeneralPurpose Serverless database
az sql db create \
--resource-group LabRG \
--server labrg-sqlserver \
--name appdb \
--edition GeneralPurpose \
--compute-model Serverless \
--family Gen5 \
--capacity 1 \
--auto-pause-delay 60 \
--max-size 2GB

# 4. Disable the public endpoint so only your Private Endpoint works
az sql server update \
--resource-group LabRG \
--name labrg-sqlserver \
--set publicNetworkAccess=Disabled


# Create the Bastion subnet
az network vnet subnet create \
--resource-group LabRG \
--vnet-name LabVNet \
--name AzureBastionSubnet \
--address-prefixes 10.1.3.0/27

# Delegate for Bastion
az network vnet subnet update \
--resource-group LabRG \
--vnet-name LabVNet \
--name AzureBastionSubnet \
--delegations Microsoft.Network/bastionHosts

# Deploy (Developer SKU) Bastion host
az network bastion create \
--resource-group LabRG \
--name LabBastionDev \
--vnet-name LabVNet \
--location centralus \
--sku Developer

# (When it errored) Delete the failed host
az network bastion delete \
--resource-group LabRG \
--name LabBastionDev \
--yes

# If that failed:
az resource delete \
--resource-group LabRG \
--namespace Microsoft.Network \
--resource-type bastionHosts \
--name LabBastionDev

# Recreate the Developer host
az network bastion create \
--resource-group LabRG \
--name LabBastionDev \
--vnet-name LabVNet \
--location centralus \
--sku Developer


az container create \
--resource-group LabRG \
--name jumpbox-aci \
--image mcr.microsoft.com/devcontainers/base:ubuntu \
--os-type Linux \
--cpu 1 \
--memory 1.5 \
--vnet LabVNet \
--subnet appSubnet \
--assign-identity \
--command-line "/bin/bash -c 'sleep infinity'" \
--restart-policy Never


az container exec \
--resource-group LabRG \
--name jumpbox-aci \
--exec-command "/bin/bash"


apt-get update
apt-get install -y curl apt-transport-https gnupg unixodbc-dev dnsutils

# Add Microsoft repo for SQL tools
curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add -
curl https://packages.microsoft.com/config/ubuntu/20.04/prod.list \
> /etc/apt/sources.list.d/mssql-release.list

apt-get update
ACCEPT_EULA=Y apt-get install -y msodbcsql17 mssql-tools

# Optional: add sqlcmd to your PATH
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
source ~/.bashrc


az container delete \
--resource-group LabRG \
--name jumpbox-aci \
--yes

