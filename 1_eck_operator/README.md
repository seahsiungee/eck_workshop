*Elastic Operator CRDs is a global resource and have already been installed by adminstrator globally on the AKS Cluster.*

# Pre-requisites
As the commands are intended to be reusable for various user with their own distinct namespace
- commands that require user specific namespaces will use `$NAMESPACE`
- kubectl apply & delete commands on yaml files with user specific namespaces will use `envsubst` to replace environment variable `$NAMESPACE` used in those files

__Set your namespace as env variable__

Replace `<yournamespace>` with your namespace e.g. `export NAMESPACE=siutest`
```
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

# Create ClusterRole & ClusterRoleBinding

**Use envsubst to replace NAMESPACE in clusterrolebinding.yaml when applying**

Execute `envsubst` command with `kubectl apply`
```
envsubst < /home/azureuser/eck_workshop/1_eck_operator/clusterrolebinding.yaml | kubectl apply -f -
```

# Enable Enterprise Trial License
```
kubectl apply -f /home/azureuser/eck_workshop/1_eck_operator/license_secret.yaml -n $NAMESPACE
```

# Get License Usage Data

```
kubectl -n $NAMESPACE get configmap elastic-licensing -o json | jq .data
```

# Troubleshooting Commands

Uninstalling elastic-operator from pre-defined namespace
```
helm uninstall elastic-operator -n <your_namespace>
```