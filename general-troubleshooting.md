## General AKS Troubleshooting

#### Environment Setup

* Deploy AKS
    * Simple setup here: [deploy-aks.md](./deploy-aks.md) steps to deploy a cluster
    * Full docs for deployment on [Azure.com](https://docs.microsoft.com/en-us/azure/aks/)
* Deploy Sample App (From [kubernetes-hackfest](https://github.com/Azure/kubernetes-hackfest/tree/bootstrapper/labs/bootstrap))
    
    ```bash
    kubectl create secret generic db-secret --from-literal=user=dbuser --from-literal=pwd=dbpassword -n hackfest
    kubectl apply -f ./app/namespace.yaml
    kubectl apply -f ./app/mongodb.yaml
    kubectl apply -f ./app/data-api.yaml
    kubectl apply -f ./app/flights-api.yaml
    kubectl apply -f ./app/quakes-api.yaml
    kubectl apply -f ./app/weather-api.yaml
    kubectl apply -f ./app/service-tracker-ui.yaml

    # browse to the service-tracker-ui Pulic IP address on port 8080 (eg - http://52.149.207.14:8080)
    ```

#### Create Chaos

* Crashing pod due to bad password in secret

    ```bash
    # re-create secret with wrong password
    kubectl delete secret db-secret -n hackfest
    kubectl create secret generic db-secret --from-literal=user=dbuser --from-literal=pwd=wrong -n hackfest

    # delete/re-create data-api deploy with new secret in place
    kubectl delete deploy data-api -n hackfest
    kubectl apply -f ./app/data-api.yaml
    ```

* Crashing pod due to memory settings

    ```bash
    kubectl apply -f ./app/quakes-api-memory.yaml
    ```

* Liveliness probe fail

    ```bash
    kubectl apply -f ./app/flights-api-health.yaml

    kubectl get pod -l app=flights-api -n hackfest
    NAME                          READY   STATUS    RESTARTS   AGE
    flights-api-57df5658c-jctcs   1/1     Running   5          85s
    ```

* Pending (scale-out, GPU selector)

    ```bash
    # create 2 jobs; one will requite a GPU nodepool
    kubectl apply -f ./app/job.yaml
    kubectl apply -f ./app/job-gpu.yaml
    ```

* Create stress on a node(s)

    ```bash     
    kubectl apply -f stress-ng.yaml

    # OR

    kubectl run -it --rm aks-stress-ng --image=ubuntu
    apt-get update && apt-get install stress-ng -y

    stress-ng --all 0 --maximize --aggressive
    stress-ng --random 64
    stress-ng --cpu 64 --cpu-method all --vm 8 --vm-bytes 80%
    stress-ng --cpu 64 --cpu-method all --vm 4 --vm-bytes 1G
    ```

#### Workload Troubleshooting

* Crashing Pods due to Config Issues

    * quakes-api

        ```bash
        # quakes-api pod in CrashLoopBackOff with Restarts
        kubectl describe pod quakes-api-57955dff5f-6jrf5 -n hackfest

        # fix by applying correct memory request/limit
        kubectl apply -f ./app/quakes-api.yaml
        ```
    
    * data-api

        ```bash
        # data-api pod seems to be running, but logs and re-starts show otherwise
        kubectl describe pod data-api-5bffdbccb4-bzqjd -n hackfest
        kubectl logs data-api-5bffdbccb4-bzqjd -n hackfest -f
        
        LOG :: CURRENT CONNECTION STRING IS "mongodb://dbuser:wrong@mongodb:27017/hackfest"
        ERROR :: CONNECTION TO DATABASE FAILED AT Tue, 12 May 2020 16:09:10 GMT
        ERROR OUTPUT :: "MongoError: Authentication failed."
        NUMBER OF CONNECTION TRIES 1
        ERROR :: CONNECTION TO DATABASE FAILED AT Tue, 12 May 2020 16:09:20 GMT
        ERROR OUTPUT :: "MongoError: Authentication failed."
        NUMBER OF CONNECTION TRIES 2

        # must fix the secret for the pod
        kubectl delete secret db-secret -n hackfest
        kubectl create secret generic db-secret --from-literal=user=dbuser --from-literal=pwd=dbpassword -n hackfest

        kubectl delete pod data-api-5bffdbccb4-bzqjd -n hackfest
        ```
    * flights-api

        ```bash
        # flights-api with numerous re-starts
        kubectl get pod -l app=flights-api -n hackfest
        NAME                          READY   STATUS    RESTARTS   AGE
        flights-api-57df5658c-jctcs   1/1     Running   5          85s

        kubectl describe pod flights-api-57df5658c-jctcs -n hackfest
        # Liveness probe failed: HTTP probe failed with statuscode: 404

        kubectl apply -f ./app/flights-api-fix.yaml -n hackfest
        ```

* Exec into pod

    ```bash
    # sometimes it helps to validate from inside the container
    kubectl get pod -n hackfest
    NAME                                  READY   STATUS    RESTARTS   AGE
    data-api-5bffdbccb4-fh8cs             1/1     Running   0          3m4s
    flights-api-77bc9d44d5-26v9d          1/1     Running   0          23h
    mongodb-768cbb678d-zvq87              1/1     Running   0          23h
    quakes-api-5685c4545d-9724l           1/1     Running   0          70s
    service-tracker-ui-5dfbf6b8b8-nqwrj   1/1     Running   0          23h
    weather-api-79786fc7c5-lplv5          1/1     Running   0          23h

    kubectl exec -it -n hackfest weather-api-79786fc7c5-lplv5 -- /bin/bash
    kubectl exec -it -n hackfest weather-api-79786fc7c5-lplv5 -- /bin/sh
    
    # then run troubleshooting commands
    $> echo $DATA_SERVICE_URI
    $> curl http://data-api.hackfest.svc.cluster.local:3009 
    ```

* Pending Pods - Scale out

    ```bash
    kubectl scale deployment service-tracker-ui -n hackfest --replicas=10

    kubectl get pod -n hackfest -l app=service-tracker-ui
    NAME                                  READY   STATUS    RESTARTS   AGE
    service-tracker-ui-5dfbf6b8b8-7qbng   0/1     Pending   0          93s
    service-tracker-ui-5dfbf6b8b8-7wt4t   1/1     Running   0          93s
    service-tracker-ui-5dfbf6b8b8-cz9k2   1/1     Running   0          93s
    service-tracker-ui-5dfbf6b8b8-dgmk5   0/1     Pending   0          93s
    service-tracker-ui-5dfbf6b8b8-hrqwg   0/1     Pending   0          93s
    service-tracker-ui-5dfbf6b8b8-jq9rq   0/1     Pending   0          93s
    service-tracker-ui-5dfbf6b8b8-lmtbn   1/1     Running   0          93s
    service-tracker-ui-5dfbf6b8b8-n8ksz   0/1     Pending   0          93s
    service-tracker-ui-5dfbf6b8b8-nqwrj   1/1     Running   0          24h
    service-tracker-ui-5dfbf6b8b8-qs427   0/1     Pending   0          93s

    kubectl get deploy service-tracker-ui -n hackfest
    NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
    service-tracker-ui   4/10    10           4           24h

    kubectl describe pod service-tracker-ui-5dfbf6b8b8-qs427 -n hackfest
    # 0/5 nodes are available: 5 Insufficient cpu.

    kubectl scale deployment service-tracker-ui -n hackfest --replicas=2
    ```

* Pending Pods - Node Selector

    ```bash
    kubectl get pod

    NAME                           READY   STATUS      RESTARTS   AGE
    image-training-job-4wz74       0/1     Completed   0          45m
    image-training-job-gpu-8wgvg   0/1     Pending     0          32m

    kubectl describe pod image-training-job-gpu-8wgvg
    # 0/5 nodes are available: 5 node(s) didn't match node selector.
    ```

* Helm

    ```bash
    # check helm version
    helm version
    version.BuildInfo{Version:"v3.2.1", GitCommit:"fe51cd1e31e6a202cba7dead9552a6d418ded79a", GitTreeState:"clean", GoVersion:"go1.13.10"}

    # try install
    helm install my-release ingress-nginx/ingress-nginx
    Error: failed to download "ingress-nginx/ingress-nginx" (hint: running `helm repo update` may help)

    # repos are missing
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm install my-release ingress-nginx/ingress-nginx
    
    # can also add main Helm catalog
    helm repo add stable https://kubernetes-charts.storage.googleapis.com

    # clean-up
    helm repo remove ingress-nginx
    helm repo remove stable
    helm delete my-release
    ```

#### Deployment Issues

* 90% of the time, AKS deployments fail due to issues with the Service Principal or subscription quota limits

* Service Principal Example: 

    ```bash
    Operation failed with status: 'Bad Request'. Details: The credentials in ServicePrincipalProfile were invalid. Please see https://aka.ms/aks-sp-help for more details. (Details: adal: Refresh request failed. Status Code = '401'. Response body: {"error":"invalid_client","error_description":"AADSTS7000215: Invalid client secret is provided.\r\nTrace ID: d690801d-405b-46a9-bf82-9ccb79b99400\r\nCorrelation ID: 8cda7b9e-90b2-48cd-9df1-a3f6edeacdca\r\nTimestamp: 2020-05-12 16:34:28Z","error_codes":[7000215],"timestamp":"2020-05-12 16:34:28Z","trace_id":"d690801d-405b-46a9-bf82-9ccb79b99400","correlation_id":"8cda7b9e-90b2-48cd-9df1-a3f6edeacdca","error_uri":"https://login.microsoftonline.com/error?code=7000215"})
    ```

* Quota Example: 

    ```bash
    az aks scale --resource-group <resource-group-name> --name <cluster-name> --node-count 5
    
    Deployment failed. Correlation ID: <correlation-id> exceeding quota limits of Core. Maximum allowed: 10, Current in use: 10, Additional requested: 2. Please read more about quota increase at http://aka.ms/corequotaincrease
    ```

* Common deployment issues are well covered in the [AKS documentation](https://docs.microsoft.com/en-us/azure/aks/troubleshooting)

#### Cluster/Node Health

* Quick Look

    ```bash
    kubectl top node
    kubectl top pod --all-namespaces
    kubectl get node -o wide
    kubectl describe node aks-nodepool1-19451884-vmss000003
    ```

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

* Control Plane (Master) Logs

    The AKS control plane logs must be setup in order to be used by customers. Docs here: https://docs.microsoft.com/en-us/azure/aks/view-master-logs

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

            # view kubelet logs
            sudo journalctl -u kubelet -o cat

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

* AKS Periscope

    AKS Periscope allows AKS customers to run initial diagnostics and collect and export the logs to help them analyze and identify potential problems or easily share the information to support to help with the troubleshooting process. More details here: https://github.com/Azure/aks-periscope

    ```bash
    az extension add --name aks-preview
    
    export CLUSTERNAME=aks-service-tracker
    export RGNAME=aks-troubleshooting
    export STORAGEACCT='/subscriptions/471d33fd-a776-405b-947c-467c291dc741/resourceGroups/monitoring-workspace/providers/Microsoft.Storage/storageAccounts/briaraksmasterlogs'

    az aks kollect -g $RGNAME -n $CLUSTERNAME --storage-account $STORAGEACCT
    ```


#### RBAC

* Is my cluster RBAC enabled? Should be the default.

    ```bash
    az aks show -g aks-troubleshooting -n aks-troubleshooting-01 -o json | grep enableRbac

    "enableRbac": true,
    ```

* Common errors

    ```bash
    kubectl --user=brian get pods
    Error from server (Forbidden): pods is forbidden: User "brian" cannot list pods in the namespace "default"
    # fix with a Role and RoleBinding
    ```

    ```yaml
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: pod-reader
    rules:
      - apiGroups: ["*"]
        resources: ["pods"]
        verbs: ["list"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: read-pods
      namespace: default
    subjects:
    - kind: User
      name: brian 
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role 
      name: pod-reader 
      apiGroup: rbac.authorization.k8s.io
    ```

* Be careful with ClusterRoles, ServiceAccounts, and the `cluster-admin` ClusterRole

    ```bash
    # this kind of thing is dangerous
    kubectl create clusterrolebinding serviceaccounts-admin --clusterrole=cluster-admin --group=system:serviceaccounts
    
    # be specific using Roles or ClusterRoles with minimal permissions and specified service accounts
    ```
#### Service Principal Issues

* The service principal used by the AKS cluster must have at least Network Contributor permissions on the subnet within your virtual network. If you wish to define a custom role instead of using the built-in Network Contributor role, the following permissions are required:
    * Microsoft.Network/virtualNetworks/subnets/join/action
    * Microsoft.Network/virtualNetworks/subnets/read

* Full details are in the [Azure Docs](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni#advanced-networking-prerequisites)
* Ideally, customers will move to [Managed Identity with AKS]()
* Example: 

    ```bash
    kubectl describe service xxxx below:

    Type     Reason                      Age              From                Message
    ----     ------                      ----             ----                -------
    Normal   EnsuringLoadBalancer        2m (x6 over 5m)  service-controller  Ensuring load balancer
    Warning  CreatingLoadBalancerFailed  2m (x6 over 5m)  service-controller  Error creating load balancer (will retry): failed to ensure load balancer for service default/servicename: network.SubnetsClient#Get: Failure responding to request: StatusCode=403 -- Original Error: autorest/azure: Service returned an error. Status=403 Code="AuthorizationFailed" Message="The client '85axxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' with object id '85axxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx' does not have authorization to perform action 'Microsoft.Network/virtualNetworks/subnets/read' over scope '/subscriptions/b8xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/resource-group-name/providers/Microsoft.Network/virtualNetworks/vnet-name/subnets/subnet-name'."
    ```

#### Image Pull Issues

* [Azure Container Registry integration](https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration)

* Private Registries (Non Azure)

    ```bash
    kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
    ```

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: private-reg
    spec:
    containers:
    - name: private-reg-container
        image: <your-private-image>
    imagePullSecrets:
    - name: regcred
    ```



