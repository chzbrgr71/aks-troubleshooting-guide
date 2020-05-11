## General AKS Troubleshooting

#### Getting started

* Use the [deploy-aks.md](./deploy-aks.md) steps to deploy a cluster
* For a sample app, use the app from the [kubernetes-hackfest](https://github.com/Azure/kubernetes-hackfest/tree/bootstrapper/labs/bootstrap)

#### Cluster/Node Health

* Azure Monitor (in Azure Portal)
* AKS Diagnostics [(Preview Feature)](https://docs.microsoft.com/en-us/azure/aks/concepts-diagnostics)
* Prometheus / Grafana

    Many customers use Prometheus and Grafana for real-time monitoring of AKS clusters. These tools can be installed as below and then used for troubleshooting

    ```bash
    # Create a new Monitoring Namespace to deploy Prometheus Operator too
    kubectl create namespace monitoring

    # Add the stable repo for Helm 3
    helm repo add stable https://kubernetes-charts.storage.googleapis.com
    helm repo update

    # Install Prometheus Operator
    helm install prometheus-operator stable/prometheus-operator --namespace monitoring

    # Check to see that all the Pods are running
    kubectl get pods -n monitoring

    # Access prometheus and grafana via port-forward (your pod names will be different)
    kubectl port-forward -n monitoring prometheus-prometheus-operator-prometheus-0 9090:9090
    kubectl port-forward -n monitoring prometheus-operator-grafana-646fd8c485-sqvf6 3000:3000
    kubectl port-forward -n monitoring alertmanager-prometheus-operator-alertmanager-0 9093:9093
    
    # Browse at http://localhost:9090/graph and http://localhost:3000
    # Username: admin 
    # Password: prom-operator
    ```

* Master and Kubelet Logs

https://docs.microsoft.com/en-us/azure/aks/view-master-logs
https://docs.microsoft.com/en-us/azure/aks/kubelet-logs

* Remote into AKS Node

    There are a few options to access AKS nodes directly. It is not recommended to enable SSH access to nodes over the internet. 

    1. Bastion Host. A bastion VM or the Azure Bastion service could be used to access nodes
    2. Helper Pod. Run a helper pod in the cluster and copy SSH keys over for access. [Documentation](https://docs.microsoft.com/en-us/azure/aks/ssh)
        * If your SSH keys were not used to create the cluster, you will follow steps in the docs to add your keys to the VM or VMSS
        * Once SSH access is obtained, use [Linux troubleshooting tools](https://docs.microsoft.com/en-us/azure/aks/troubleshoot-linux) as needed. 

            ```bash
            kubectl run --generator=run-pod/v1 -it --rm aks-ssh --image=debian
            apt-get update && apt-get install openssh-client -y

            # in a seperate terminal window: 
            kubectl cp ~/.ssh/id_rsa $(kubectl get pod -l run=aks-ssh -o jsonpath='{.items[0].metadata.name}'):/id_rsa

            # back at Run window
            chmod 0600 id_rsa
            ssh -i id_rsa azureuser@10.240.0.4

            # some sample commands
            top
            dmesg | tail
            mpstat -P ALL
            free -m
            iostat
            netstat
            ```

* Exec into troubleshooting container (instead of ssh)

    Some customers keep a utility image with troubleshooting tools installed. In some cases, these pods will need to be run in privledged mode. 

    ```bash
    # run this pod and note that Alpine images do not have tools such as curl, nmap, tcptraceroute, etc.
    kubectl run --generator=run-pod/v1 -it --rm no-tools --image=alpine

    # now run this pod and see that numerous tools can be added in advance as needed
    kubectl run --generator=run-pod/v1 -it --rm aks-network-tools --image=nicolaka/netshoot
    # https://github.com/nicolaka/netshoot 
    ```

* AKS Periscope. https://github.com/Azure/aks-periscope
* Review Common Issues in the [AKS documentation](https://docs.microsoft.com/en-us/azure/aks/troubleshooting) 


#### Workload Troubleshooting

Exec into pod
GPU's
Helm

#### Resource Requests/Limits


#### RBAC


#### Service Principal Issues


#### Image Pull Issues


#### Storage

