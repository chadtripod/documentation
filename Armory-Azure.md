az login
az account list
SUBSCRIPTION_ID=<SUBSCIPTION_ID>
az account set --subscription @SUBSCRIPTION_ID
az ad sp create-for-rbac --name "Spinnaker-SP"
APP_ID=<APP_ID>
TENANT_ID=<TENANT_ID>
az account list-locations --query [].name
az group create --name <resourcegroup_name> --location <location>
RESOURCE_GROUP=SPINNAKER-RG
i.e. - az group create --name $RESOURCE_GROUP --location us-east
VAULT_NAME=Spinnaker-Vault
az keyvault create --enabled-for-template-deployment true --resource-group @RESOURCE_GROUP --name $VAULT_NAME
az keyvault set-policy --secret-permissions get --name $VAULT_NAME --spn $APP_ID
VMUSER=<USERNAME>
az keyvault secret set --name VMUsername --value $VMUSER

#generate ssh pubkey to insert into vault
ssh-keygen -f <user> -N ''
VMUSER_KEY="ssh-rsa...."
az keyvault secrets set --name VMSSHPUBLICKEY --valut-name $VAULT_NAME --value "$VMUSER_KEY"

#ADD CONFIGURATION TO HALYARD

SUBSCRIPTION_ID=
APP_ID=
TENANT_ID=
RESOURCE_GROUP=
VAULT_NAME=

hal config provider azure account add spin-azure-account --client-id $APP_ID --tenant-id $TENANT_ID --subscription-id $SUBSCRIPTION_ID --default-key-vault $VAULT_NAME --default-resource-group $RESOURCE_GROUP --packer-resource-group $RESOURCE_GROUP --packer-resource-group $RESOURCE_GROUP --app-key
hal config provider azure enable
hal config provider azure account list
hal config provider azure account get <account_name>
hal config provider azure account edit <account_name> --regions=westus
hal deploy apply
