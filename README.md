# IBM CloudPaks on Azure ARO



## Deploying Azure ARO

The following QuickStart guide is based on [Microsoft's Tutorial](https://docs.microsoft.com/en-us/azure/openshift/tutorial-create-cluster) on setting up OpenShift ARO



### Permissions

Verify the account deploying ARO has the `User Access Administrator` and `Contributor` permissions.



### Subscription One-Time Setup

If needed, downlaod the azure command line utility.  You will need version 2.6.0 or later.

```bash
# On macOS
$ brew update
$ brew install azure-cli

# on Linux
$ curl -L https://aka.ms/InstallAzureCli | bash
```



Log on to your azure subscription thru the `az` cli.

```bash
# login to your azure subscription
$ az login

# If you have more than one subscription, set the subscription you want to use
$ az account set --subscription <SUBSCRIPTION ID>
```



Register the `Microsoft.RedHatOpenShift`, `Microsoft.Compute` and `Microsoft.Storage` resource providers

```bash
$ az provider register -n Microsoft.RedHatOpenShift --wait
$ az provider register -n Microsoft.Compute --wait
$ az provider register -n Microsoft.Storage --wait
```



### Environment Variables

The following variables are used for this QuickStart guide.

```bash
$ LOCATION=eastus                 # the location of your cluster
$ CLUSTER=cp4x                    # the name of your cluster
$ RESOURCEGROUP=aro-${CLUSTER}-rg # the name of the resource group where you want to create your cluster
```



### Create Infrastructure

We will need to create a resource group.  This resource group a single vnet, which in turn will contain two subnets, one for master nodes and one for worker nodes

```bash
# create a resource group
$ az group create --name $RESOURCEGROUP \
	--location $LOCATION

# create a vnet
$ az network vnet create --resource-group $RESOURCEGROUP \
	--name $CLUSTER-vnet \
	--address-prefixes 10.0.0.0/22

# create a master subnet
$ az network vnet subnet create --resource-group $RESOURCEGROUP \
	--vnet-name $CLUSTER-vnet \
	--name master-subnet \
	--address-prefixes 10.0.0.0/23 \
	--service-endpoints Microsoft.ContainerRegistry

# create a worker subnet
$ az network vnet subnet create --resource-group $RESOURCEGROUP \
	--vnet-name $CLUSTER-vnet \
	--name worker-subnet \
	--address-prefixes 10.0.2.0/23 \
	--service-endpoints Microsoft.ContainerRegistry

# disable subnet private endpoint policies
$ az network vnet subnet update --name master-subnet \
	--resource-group $RESOURCEGROUP \
	--vnet-name $CLUSTER-vnet \
	--disable-private-link-service-network-policies true

# create a service principal
$ az ad sp create-for-rbac --name $CLUSTER-aro-service-principal \
  --role Contributor > serviceprincipal.json

# obtain AppID and password for service principal
$ SP_APPID=$(jq -r .appId serviceprincipal.json)
$ SP_PASSWD=$(jq -r '.password' serviceprincipal.json)
```



### Create the ARO Cluster

When creating a cluster, you can optionally pass your OpenShift Pull Secret to the installer which enables your cluster to access Red Hat container registries along with additional content. Add the `--pull-secret @pull-secret.txt` argument to your command.  If you want to use a custom domain, add  `--domain foo.example.com` argument to your command, replacing `foo.example.com` with your own custom domain.  See [using a custom domain](#using-a-custom-domain) for implementation details. You can set the `--ingress-visibility` and `--apiserver-visibility` parameters to `Private` to restrict acess to your cluster.  Use [this link](https://docs.microsoft.com/en-us/azure/virtual-machines/sizes) for a list of available VM sizes and their characteristics.

```bash
$ az aro create --resource-group $RESOURCEGROUP --name $CLUSTER --vnet $CLUSTER-vnet \
	--client-id ${SP_APPID} --client-secret ${SP_PASSWD} \
	--master-subnet master-subnet --worker-subnet worker-subnet \
	--master-vm-size Standard_D4s_v3 --worker-vm-size Standard_D4s_v3 --worker-count 3 \
	--pull-secret @pull-secret.txt --domain foo.example.com \
	--apiserver-visibility Public --ingress-visibility Public
```

This step can take up to 45 minutes to complete.



### Accessing your cluster

```bash
# obtain your openshift console URL
$ az aro show --name $CLUSTER --resource-group $RESOURCEGROUP --query "consoleProfile.url" -o tsv

# obtain your openshift api endpoint
$ az aro show --name $CLUSTER --resource-group $RESOURCEGROUP --query "apiserverProfile.url" -o tsv

# obtain your kubeadmin password
$ az aro list-credentials --name $CLUSTER --resource-group $RESOURCEGROUP --query "kubeadminPassword" -o tsv

```



### Using a custom domain

If you use a custom domain you must create 2 DNS A records in your DNS server for the `--domain` specified:

- **api** - points to the API Server
- ***.apps** - points to the Ingress

Once your cluster is up, run the following commands and update your DNS server

```bash
# obtain IP Address for API Server
$ az aro show --name $CLUSTER --resource-group $RESOURCEGROUP --query "apiserverProfile.ip" -o tsv

# obtain IP Address for Ingress
$ az aro show --name $CLUSTER --resource-group $RESOURCEGROUP --query "ingressProfiles[0].ip" -o tsv
```



### Creating a ReadWriteMany storage class

By default, your ARO cluster will contain the storage class `managed-premium` that can dynamically provision ReadWriteOnce PersistentVolumes using `azure-disk`. If your CloudPak requires a RWX storage class, you can create one using `azure-file`  with the following process:

```bash
# obtain the managed resource group for your ARO cluster
# WARNING: If you set a --domain when deploying ARO, the value of clusterProfile.domain
# 	will be your domain.  Log on to your azure portal and manually set the MANAGED_RG variable
$ MANAGED_RG="aro-$(az aro show --name $CLUSTER \
	--resource-group $RESOURCEGROUP --query "clusterProfile.domain" -o tsv)"

# obtain the storage account that can be used to create azure-file storage
$ MANAGED_SA=$(az storage account list \
	--resource-group $MANAGED_RG --query "[?starts_with(name, '$CLUSTER')].name" -o tsv)

# log on to your openshift cluster via command line
$ APISERVER=$(az aro show --name $CLUSTER \
	--resource-group $RESOURCEGROUP --query "apiserverProfile.url" -o tsv)
$ PASSWORD=$(az aro list-credentials --name $CLUSTER \
	--resource-group $RESOURCEGROUP --query "kubeadminPassword" -o tsv)
$ oc login $APISERVER -u kubeadmin -p $PASSWORD --insecure-skip-tls-verify

# create ClusterRole to access secrets
$ cat << EOF | oc create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:azure-file-volume-binder
rules:
- apiGroups: ['']
  resources: ['secrets']
  verbs:     ['get','create']
EOF

# assign ClusterRole to ServiceAccount
$ oc adm policy add-cluster-role-to-user \
	system:azure-file-volume-binder \
	system:serviceaccount:kube-system:persistent-volume-binder

# create the azure-file storage class
$ cat << EOF | oc create -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azure-file
provisioner: kubernetes.io/azure-file
parameters:
  storageAccount: ${MANAGED_SA}
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```



### Integrating with Azure AD

```bash
# setup environment
$ SP_AAD_PASSWORD='Y0urS3cr3tPa55w@rd'
$ AZURE_TENNANT_ID="$(az account show --query tenantId -o tsv)
$ CONSOLE_URL=$(az aro show --name $CLUSTER --resource-group $RESOURCEGROUP --query "consoleProfile.url" -o tsv)
$ OAUTH_ENDPOINT=$(echo $CONSOLE_URL|sed 's/console-openshift-console/oauth-openshift/')

# log on to your openshift cluster via command line
$ APISERVER=$(az aro show --name $CLUSTER --resource-group $RESOURCEGROUP --query "apiserverProfile.url" -o tsv)
$ PASSWORD=$(az aro list-credentials --name $CLUSTER --resource-group $RESOURCEGROUP --query "kubeadminPassword" -o tsv)
$ oc login $APISERVER -u kubeadmin -p $PASSWORD --insecure-skip-tls-verify

# create a secret for Azure AD integration
$ oc create secret generic openid-client-secret-azuread \
	--from-literal=clientSecret=${SP_AAD_PASSWORD} -n openshift-config

# create an App Registration with callbacks to your console URL
$ AZUREAD_APPID=$(az ad app create --display-name $CLUSTER-azuread-auth \
	--reply-urls ${OAUTH_ENDPOINT}/oauth2callback/AAD \
	--homepage ${CONSOLE_URL} \
	--identifier-uris ${CONSOLE_URL} \
	--password ${SP_AAD_PASSWORD} \
	--query appId -o tsv)

$ cat > manifest.json<< EOF
[{
  "name": "upn",
  "source": null,
  "essential": false,
  "additionalProperties": []
},
{
"name": "email",
  "source": null,
  "essential": false,
  "additionalProperties": []
}]
EOF

# update app optionalClaims
$ az ad app update --set optionalClaims.idToken=@manifest.json --id ${AZUREAD_APPID}

# update AAD app scope permissions
# WARNING: Unless you are authenticated as a Global Administrator for this Azure Active Directory
# 	you can ignore the message to grant the consent, since you'll be asked to do this 
# 	once you login on your own account.
$ az ad app permission add \
 --api 00000002-0000-0000-c000-000000000000 \
 --api-permissions 311a71cc-e848-46a1-bdf8-97ff7156d8e6=Scope \
 --id ${AZUREAD_APPID}

# update OpenShift OAuth
$ cat << EOF | oc replace -f -
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: AAD
    mappingMethod: claim
    type: OpenID
    openID:
      clientID: ${AZUREAD_APPID}
      clientSecret:
        name: openid-client-secret-azuread
      extraScopes:
      - email
      - profile
      extraAuthorizeParameters:
        include_granted_scopes: "true"
      claims:
        preferredUsername:
        - email
        - upn
        name:
        - name
        email:
        - email
      issuer: https://login.microsoftonline.com/${AZURE_TENNANT_ID}
EOF
```



## Cleanup

Remove your cluster and all associated resources from your Azure subscription
```bash
$ az aro delete -g $RESOURCEGROUP -n $CLUSTER -y
$ az ad app delete --id $AZUREAD_APPID
$ az ad sp delete --id $SP_APPID
$ az group delete -g $RESOURCEGROUP --no-wait -y
```

