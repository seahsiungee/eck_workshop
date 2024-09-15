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

# Install ECK Operator via Helm Chart for pre-defined namespace
```
helm install elastic-operator elastic/eck-operator --version 2.13.0 -n $NAMESPACE --create-namespace \
  --set=installCRDs=false \
  --set=managedNamespaces='{$NAMESPACE}' \
  --set=createClusterScopedResources=false \
  --set=webhook.enabled=false \
  --set=config.validateStorageClass=false \
  --set=operator-namespace=$NAMESPACE \
  --set=metrics-port=6060
```

Check that operator is running in your namespace
```
kubectl get sts -A -o wide --field-selector metadata.namespace=$NAMESPACE
```

You should see 
```bash
azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl get sts -A -o wide --field-selector metadata.namespace=$NAMESPACE
NAMESPACE   NAME                       READY   AGE    CONTAINERS      IMAGES
siu         elastic-operator           1/1     101m   manager         docker.elastic.co/eck/eck-operator:2.13.0
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