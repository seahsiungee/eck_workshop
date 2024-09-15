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
  --set=managedNamespaces={$NAMESPACE} \
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

Useful Commands
```
# View configmap of elastic-operator i.e. configuration of your elastic-operator
kubectl describe cm/elastic-operator -n $NAMESPACE
```

# Create ClusterRole & ClusterRoleBinding

**Use envsubst to replace NAMESPACE in clusterrolebinding.yaml when applying**

Execute `envsubst` command with `kubectl apply`
```
envsubst < /home/azureuser/eck_workshop/1_eck_operator/clusterrolebinding.yaml | kubectl apply -f -
```

Check that ClusterRole & ClusterRoleBinding are created correctly with your namespace prefixed
```
kubectl get clusterrole,clusterrolebinding | grep $NAMESPACE
```

You should see 
```bash
azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl get clusterrole,clusterrolebinding | grep $NAMESPACE
clusterrole.rbac.authorization.k8s.io/siu-elastic-operator                                                      2024-09-12T13:54:40Z
clusterrolebinding.rbac.authorization.k8s.io/siu-elastic-operator                                                             ClusterRole/siu-elastic-operator                                                                ClusterRole/siu-elastic-operator     
```

# Enable Enterprise Trial License
```
kubectl apply -f /home/azureuser/eck_workshop/1_eck_operator/license_secret.yaml -n $NAMESPACE
```

Check that license secret was created correctly with your namespace
```bash
kubectl get secrets -A --field-selector metadata.namespace=$NAMESPACE
```

You should see
```bash
azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl get secrets -A --field-selector metadata.namespace=$NAMESPACE
NAMESPACE   NAME                                          TYPE                 DATA   AGE
siu         eck-trial-license                             Opaque               1      103m
```

# Get License Usage Data

```
kubectl -n $NAMESPACE get configmap elastic-licensing -o json | jq .data
```

You should see
```bash
azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl -n $NAMESPACE get configmap elastic-licensing -o json | jq .data
{
  "eck_license_expiry_date": "2024-10-13T16:34:25Z",
  "eck_license_level": "enterprise_trial",
  "enterprise_resource_units": "0",
  "timestamp": "2024-09-13T16:34:43Z",
  "total_managed_memory": "0.00GiB",
  "total_managed_memory_bytes": "0"
}
```

# Troubleshooting Commands

Uninstalling elastic-operator from pre-defined namespace
```
helm uninstall elastic-operator -n $NAMESPACE
```