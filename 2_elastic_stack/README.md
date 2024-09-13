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

# Tasks
## 1. Deploy 3 node elasticsearch cluster + Kibana
### Deploy a 3 node elasticsearch cluster
```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/elasticsearch.yaml -n $NAMESPACE
```

Get password for elastic
```
kubectl get secret elasticsearch-es-elastic-user -n $NAMESPACE -o go-template='{{.data.elastic | base64decode}}'
```

### Deploy kibana referencing the elasticsearch cluster created above
```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/kibana.yaml -n $NAMESPACE
```

### Access Kibana and list Elasticsearch Nodes
1. Get Public IP to access Kibana
    ```
    kubectl get svc -n $NAMESPACE
    ```

2. Navigate to Kibana > Management > Dev Tools

    Execute the following command in Dev Tools
    ```
    GET _cat/nodes?v&s=name
    ```
    You should see a response similar to the below
    ```
    ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
    10.244.3.113           22          76  11    0.00    0.03     0.06 cdfhilmrstw *      elasticsearch-es-default-0
    10.244.4.70            41          76  14    0.09    0.16     0.16 cdfhilmrstw -      elasticsearch-es-default-1
    10.244.2.92            49          75   8    0.23    0.43     0.27 cdfhilmrstw -      elasticsearch-es-default-2
    ```

## 2. Update Elasticsearch to Hot-Frozen Topology

```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/elasticsearch_hot_frozen.yaml -n $NAMESPACE
```

## 3. Update deployment to enable monitoring
Elasticsearch Cluster
```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/elasticsearch_hot_frozen_monitored.yaml -n $NAMESPACE
```

Kibana
```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/kibana_monitored.yaml -n $NAMESPACE
```
