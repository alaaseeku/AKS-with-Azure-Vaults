# AKS-with-Azure-Vaults

#Create Csi driver using helm charts
helm repo add secrets-store-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/master/charts
kubectl create ns $CSI_NS
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver -n $CSI_NS

kubectl apply -f https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/deployment/provider-azure-installer.yaml --namespace $CSI_NS
#create Azure Vault
az keyvault create -n $VaultName -g $resourceGroup  
#create Azure Vault secret
az keyvault secret set --name $secretName --value $secretValue --vault-name $VaultName

#Connect Csi driver to the Azure vault secret 

$secretProviderKV = @"
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: azure-kv
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"   # We will not use pod identity for this example. We will use SP
    useVMManagedIdentity: "false"
    userAssignedIdentityID: ""
    keyvaultName: $VaultName   # This is the name of KeyVault resource that we created in previous step 
    objects:  |
      array:
        - |
          objectName: $secretName
          objectType: secret      # object types: secret, key or cert
          objectVersion: "" 
       
    resourceGroup: $resourceGroup    # Resource goup that you have used to create KeyVault
    subscriptionId: $SUBID       
    tenantId: $TenantID     
"@
$secretProviderKV | kubectl create -f -
#Create the required authentication and access policy
# Create SP 
az ad sp create-for-rbac --skip-assignment --name $SP_NAME
#save displayed Client password ($SP_CLIENT_PW)
# Get client id of SP
$SP_CLIENT_ID=$(az ad sp show --id http://$SP_NAME --query appId -o tsv)
# Role Assignment for AKV
az role assignment create --role Reader --assignee $SP_CLIENT_ID --scope /subscriptions/$SUBID/resourcegroups/$resourceGroup/providers/Microsoft.KeyVault/vaults/$VaultName
# Set permissions to read secrets
az keyvault set-policy -n $VaultName --secret-permissions get --spn $SP_CLIENT_ID

kubectl create secret generic $SecretVolName  --from-literal clientid=$SP_CLIENT_ID --from-literal clientsecret=$SP_CLIENT_PW

# create a test pod 
kubectl apply -f sec.yaml
#Show secrets
kubectl exec -it nginx-secrets-store ls /mnt/secrets-store/
# Verify secrets mentioned in provider class
kubectl exec -it nginx-secrets-store cat /mnt/secrets-store/db-username
kubectl exec -it nginx-secrets-store cat /mnt/secrets-store/db-password
