# Info
Common assets e.g. centralised stack monitoring deployment are intended to be deployed in a pre-defined namespace, so all commands and yaml in this folder will use its own intended namespaces.

# Instructions

## Centralised Stack Monitoring Namespace & Deployment Setup
To be deployed into `stack-mon` Namespace, which is defined in the yaml.

```
# Install eck operator
helm install elastic-operator elastic/eck-operator --version 2.13.0 -n stack-mon --create-namespace \
  --set=installCRDs=false \
  --set=managedNamespaces='{stack-mon}' \
  --set=createClusterScopedResources=false \
  --set=webhook.enabled=false \
  --set=config.validateStorageClass=false \
  --set=operator-namespace=stack-mon

# Install create clusterrole and clusterrolebinding
kubectl apply -f /home/azureuser/eck_workshop/common/clusterrolebinding.yaml -n stack-mon

# Install create central_monitoring Elasticsearch and Kibana
kubectl apply -f /home/azureuser/eck_workshop/common/central_monitoring_elasticsearch.yaml
kubectl apply -f /home/azureuser/eck_workshop/common/central_monitoring_kibana.yaml

# Get Password for elastic user
kubectl get secret central-stack-mon-es-elastic-user -n stack-mon -o go-template='{{.data.elastic | base64decode}}'
```