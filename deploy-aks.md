## Deploying demo AKS for labs

```bash
# modify below for your Azure sub
export CLUSTERNAME=aks-troubleshooting
export K8SVERSION=1.15.10
export VMSIZE=Standard_D2_v2
export NODECOUNT=5
export RGNAME=aks-troubleshooting
export LOCATION=eastus
export CLIENTID=<replace-me>
export CLIENTSECRET=<replace-me>
export AZUREMONITOR=/subscriptions/<replace-me>/resourcegroups/<replace-me>/providers/microsoft.operationalinsights/workspaces/<replace-me>

az group create --name $RGNAME --location $LOCATION

az aks create \
    --resource-group $RGNAME \
    --name $CLUSTERNAME \
    --node-count $NODECOUNT \
    --kubernetes-version $K8SVERSION \
    --service-principal $CLIENTID \
    --client-secret $CLIENTSECRET \
    --workspace-resource-id $AZUREMONITOR \
    --enable-addons monitoring \
    --no-wait

az aks get-credentials -n $CLUSTERNAME -g $RGNAME
```
