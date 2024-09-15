*Elastic Operator CRDs is a global resource and have already been installed by adminstrator globally on the AKS Cluster.*

# Pre-requisites
As the commands are intended to be reusable for various user with their own distinct namespace
- commands that require user specific namespaces will use `$NAMESPACE`
- kubectl apply & delete commands on yaml files with user specific namespaces will use `envsubst` to replace environment variable `$NAMESPACE` used in those files

__Set your namespace as env variable__

Replace `<yournamespace>` with your namespace e.g. `export NAMESPACE=siutest`
```bash
export NAMESPACE=<yournamespace>
```
Remember to set your namespaces as env variable everytime you use a new bash session.

As multiple users are using the same vm-bastion with `azureuser` user, please **DO NOT set your `NAMESPACE` env variable in the `/home/azureuser/.bashrc`**

# Upgrade your Elastic Stack Deployment to v8.15.1

Apply prepared Elasticsearch Cluster manifest with version specified as 8.15.1
```
kubectl apply -f /home/azureuser/eck_workshop/3_eck_day2_ops/elasticsearch_hot_frozen_monitored_snapshot_8151.yaml -n $NAMESPACE
```

Check node version from rolling upgrade in Kibana DevTools
```
GET _cat/nodes?v&h=ip,heap.percent,ram.percent,cpu,node.role,master,name,version&s=node.role,version
```

While rolling upgrade is progressing, you should see
```
ip           heap.percent ram.percent cpu node.role master name                      version
10.244.5.43            28          73  46 f         -      elasticsearch-es-frozen-0 8.15.1
10.244.3.255           40          78  15 hirst     -      elasticsearch-es-hot-0    8.15.0
10.244.6.60            30          86   1 m         -      elasticsearch-es-master-1 8.15.0
10.244.8.249           38          86   5 m         -      elasticsearch-es-master-2 8.15.0
10.244.7.18            45          88   3 m         *      elasticsearch-es-master-0 8.15.0
```
node version being upgraded based on expected order by node role
```
10.244.5.43            14          73   3 f         -      elasticsearch-es-frozen-0 8.15.1
10.244.3.255           44          78   5 hirst     -      elasticsearch-es-hot-0    8.15.0
10.244.4.151           15          73  14 hirst     -      elasticsearch-es-hot-1    8.15.1
10.244.8.249           17          86   2 m         -      elasticsearch-es-master-2 8.15.0
10.244.7.18            38          88   5 m         *      elasticsearch-es-master-0 8.15.0
10.244.6.60            59          86   3 m         -      elasticsearch-es-master-1 8.15.0
```

# Controlling restarts during upgrade of ECK Operator 

List all elastic resources with annotation `eck.k8s.elastic.co/managed=false`
```
kubectl get pod -A \
-o jsonpath='{range .items[?(@.metadata.annotations.eck\.k8s\.elastic\.co/managed)]}{.metadata.name}{"\t"}{.metadata.annotations.eck\.k8s\.elastic\.co/managed}{"\t"}{.metadata.namespace}{"\n"}'
```

Annotate all elastic resource in your namespace with `eck.k8s.elastic.co/managed=false`
```
ANNOTATION='eck.k8s.elastic.co/managed=false'
kubectl annotate --overwrite elastic --all -n $NAMESPACE $ANNOTATION
```

Upgrade ECK Operator via Helm Chart in your namespace
```
helm upgrade elastic-operator elastic/eck-operator --version 2.14.0 -n $NAMESPACE 
```

Check that operator is running in your namespace
```
kubectl get sts -A -o wide --field-selector metadata.namespace=$NAMESPACE
```

You should see 
```bash
azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl get sts -A -o wide --field-selector metadata.namespace=$NAMESPACE
NAMESPACE   NAME                       READY   AGE    CONTAINERS      IMAGES
siu         elastic-operator           1/1     101m   manager         docker.elastic.co/eck/eck-operator:2.14.0
```

Resume Elastic resource management by operator
```
RM_ANNOTATION='eck.k8s.elastic.co/managed-'

# Resume management of a single Elasticsearch cluster named "quickstart"
kubectl annotate --overwrite elastic --all -n $NAMESPACE $RM_ANNOTATION
```

# Generate ECK Diagnostics

ECK Diagnostics tool has already been downloaded onto vm-bastion at the following location `/home/azureuser/eck_diagnostics/`

Execute the following command to generate default set of diagnostics for your own namespace
```
/home/azureuser/eck_diagnostics/eck-diagnostics -o $NAMESPACE -r $NAMESPACE --output-directory /home/azureuser/eck_diagnostics/generated_diags/ -n $NAMESPACE-eck-diagnostics-$(date +%F).zip
```

You should see
```bash
azureuser@siu-scb-wks-bastion:~$ /home/azureuser/eck_diagnostics/eck-diagnostics -o $NAMESPACE -r $NAMESPACE --output-directory /home/azureuser/eck_diagnostics/generated_diags/ -n $NAMESPACE-eck-diagnostics-$(date +%F).zip
2024/09/15 16:39:51 ECK diagnostics with parameters: {DiagnosticImage:docker.elastic.co/eck-dev/support-diagnostics:8.5.0 ECKVersion: Kubeconfig: OperatorNamespaces:[siu] ResourcesNamespaces:[siu] OutputDir:/home/azureuser/eck_diagnostics/generated_diags/ OutputName:siu-eck-diagnostics-2024-09-15.zip RunStackDiagnostics:true RunAgentDiagnostics:false Verbose:false StackDiagnosticsTimeout:5m0s Filters:map[] LogFilters:map[]}
2024/09/15 16:39:52 ECK version is 2.13.0
2024/09/15 16:39:52 Extracting Kubernetes diagnostics from siu
2024/09/15 16:40:05 Elasticsearch diagnostics extracted for siu/elasticsearch
2024/09/15 16:40:05 Kibana diagnostics extracted for siu/kibana
2024/09/15 16:40:05 ECK diagnostics written to /home/azureuser/eck_diagnostics/generated_diags/siu-eck-diagnostics-2024-09-15.zip
```