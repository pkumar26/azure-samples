## AKS Cluster in your own VNet (Kubenet + Calico)

This configures AKS cluster to use Calico CNI with Host-Local IPAM and Calico network policy engine.

\
**Setup environment variables**

    RG_NAME=""
    AZ_REGION=""
    VNET_NAME=""
    VNET_PREFIX=""
    SUBNET1_NAME=""
    SUBNET1_PREFIX=""
    SUBNET2_NAME=""  ## Optional
    SUBNET2_PREFIX="" ## Optional
    CLUSTER_NAME=""
    K8S_VERSION=""
    EGRESS_PUBIP="" ## Use if you want to give your cluster a static IP for outbound access
    NETWORK_PLUGIN="kubenet"
    NETWORK_POLICY="calico"
    ACR_NAME=""
    SP_NAME=""
    VM_SIZE=""
<!--
    LA_WORKSPACE_NAME=""
    LA_WORKSPACE_RG=""
-->

**Get supported k8s versions**

    az aks get-versions --location $RG_REGION --output table

**Create a resource group**

    az group create --name $RG_NAME --location $AZ_REGION

**Create a virtual network and subnets**

    az network vnet create \
        --name $VNET_NAME \
        --address-prefixes $VNET_PREFIX \
        --resource-group $RG_NAME
    az network vnet subnet create \
        --name "$SUBNET1_NAME" \
        --address-prefixes "$SUBNET1_PREFIX" \
        --resource-group $RG_NAME \
        --vnet-name $VNET_NAME
    az network vnet subnet create \
        --name "$SUBNET2_NAME" \
        --address-prefixes "$SUBNET2_PREFIX" \
        --resource-group $RG_NAME \
        --vnet-name $VNET_NAME

**Create PublcIP for Egress**

    az network public-ip create \
        --name $EGRESS_PUBIP \
        --sku standard \
        --resource-group $RG_NAME
>If Public IPs are not allowed for you and your cluster sits behind Network Firewall, please look [here](https://docs.microsoft.com/en-us/azure/aks/egress-outboundtype) and work with your networking team for custom setup.

**Get ID of PublicIP created above**

    EGRESS_ID=$(az network public-ip show --resource-group $RG_NAME --name $EGRESS_PUBIP --query id -o tsv)

**Create a service principal and read in the application ID**

>If you decide to use Managed Identity, then you don't need to create Service Principal. Just pass --enable-managed-identity while cluster creation

    SP=$(az ad sp create-for-rbac -n $SP_NAME --skip-assignment)
    SP_ID=$(echo $SP | jq -r .appId)
    SP_PASSWORD=$(echo $SP | jq -r .password)

>Wait 15 seconds to make sure that service principal has propagated
    echo "Waiting for service principal to propagate..."
    sleep 15

**Get the virtual network resource ID**

    VNET_ID=$(az network vnet show --resource-group $RG_NAME --name $VNET_NAME --query id -o tsv)

**Assign the service principal Owner permissions to the PublicIP resource**

    az role assignment create --assignee $SP_ID --scope $EGRESS_ID --role Owner

**Assign the service principal Owner permissions to the virtual network resource**

    az role assignment create --assignee $SP_ID --scope $VNET_ID --role Owner

**Get the virtual network subnet resource ID**

    SUBNET1_ID=$(az network vnet subnet show --resource-group $RG_NAME --vnet-name $VNET_NAME --name $SUBNET1_NAME --query id -o tsv)
    SUBNET2_ID=$(az network vnet subnet show --resource-group $RG_NAME --vnet-name $VNET_NAME --name $SUBNET2_NAME --query id -o tsv)

<!--
**Get Log analytics Workspace ID URL**

    LA_WORKSPACE_ID=$(az monitor log-analytics workspace show --resource-group $LA_WORKSPACE_RG --workspace-name $LA_WORKSPACE_NAME --query id -o tsv)
-->

**Create the AKS cluster and specify the virtual network and service principal information**

    az aks create \
        --name $CLUSTER_NAME \
        --resource-group $RG_NAME \
        --kubernetes-version $K8S_VERSION \
        --vm-set-type VirtualMachineScaleSets \
        --node-vm-size $VM_SIZE \
        --node-count 3 \
        --zones 1 2 3 \
        --nodepool-name nodepool00 \
        --enable-cluster-autoscaler \
        --generate-ssh-keys \
        --min-count 3 \
        --max-count 6 \
        --load-balancer-sku standard \
        --load-balancer-outbound-ips $EGRESS_ID \
        --network-plugin $NETWORK_PLUGIN \
        --network-policy $NETWORK_POLICY \
        --service-cidr 10.0.0.0/16 \
        --dns-service-ip 10.0.0.10 \
        --docker-bridge-address 172.17.0.1/16 \
        --vnet-subnet-id $SUBNET1_ID \
        --service-principal $SP_ID \
        --client-secret $SP_PASSWORD \
        --attach-acr $ACR_NAME
<!--
        --enable-addons monitoring \
        --workspace-resource-id $LA_WORKSPACE_ID
-->
>--zones will work in regions where zones are supported. Make sure that your region support [zones](https://docs.microsoft.com/en-us/azure/availability-zones/az-region)

**Get credentials to login**

    az aks get-credentials --resource-group $RG_NAME --name $CLUSTER_NAME
<!--
**Create ClusterRoleBinding before you can correctly access the dashboard**

    kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard

**Browse AKS dashboard**

    az aks browse --resource-group $RG_NAME --name $CLUSTER_NAME
-->

**Chck the status of AKS worker nodes**

    kubectl get nodes -o wide

**Verify outgoing IP of your cluster**

    kubectl run -it --rm aks-ip --image=debian --generator=run-pod/v1
    apt-get update && apt-get install curl -y
    curl -s checkip.dyndns.org
>IP should match the EGRESS IP of the cluster
