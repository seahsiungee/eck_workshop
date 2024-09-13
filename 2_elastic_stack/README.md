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

Create Elasticsearch Cluster

```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/elasticsearch.yaml -n $NAMESPACE
```

Check that statefulset & pods are running
```
kubectl get sts,pods -A --field-selector metadata.namespace=$NAMESPACE
```

You should see 
```bash
azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl get sts,pods -A --field-selector metadata.namespace=$NAMESPACE
NAMESPACE   NAME                                        READY   AGE
siu         statefulset.apps/elastic-operator           1/1     107m
siu         statefulset.apps/elasticsearch-es-default   3/3     105m

NAMESPACE   NAME                             READY   STATUS    RESTARTS   AGE
siu         pod/elastic-operator-0           1/1     Running   0          107m
siu         pod/elasticsearch-es-default-0   1/1     Running   0          105m
siu         pod/elasticsearch-es-default-1   1/1     Running   0          105m
siu         pod/elasticsearch-es-default-2   1/1     Running   0          105m
```

### Deploy kibana referencing the elasticsearch cluster created above
```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/kibana.yaml -n $NAMESPACE
```

Check that deployment & pods are running
```
kubectl get deploy,pods -A --field-selector metadata.namespace=$NAMESPACE
```

You should see 
```bash
azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl get deploy,pods -A --field-selector metadata.namespace=$NAMESPACE
NAMESPACE   NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
siu         deployment.apps/kibana-kb   1/1     1            1           48m

NAMESPACE   NAME                             READY   STATUS    RESTARTS   AGE
siu         pod/elastic-operator-0           1/1     Running   0          108m
siu         pod/elasticsearch-es-default-0   1/1     Running   0          106m
siu         pod/elasticsearch-es-default-1   1/1     Running   0          106m
siu         pod/elasticsearch-es-default-2   1/1     Running   0          106m
siu         pod/kibana-kb-6b6f45bcbb-krk2t   1/1     Running   0          48m
```

### Access Kibana and list Elasticsearch Nodes
1. Get Public IP to access Kibana
    ```
    kubectl get svc -n $NAMESPACE
    ```

    You should see
    ```bash
    azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl get svc -n $NAMESPACE
    NAME                             TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
    elasticsearch-es-default         ClusterIP      None           <none>           9200/TCP         103m
    elasticsearch-es-http            ClusterIP      10.0.191.103   <none>           9200/TCP         103m
    elasticsearch-es-internal-http   ClusterIP      10.0.88.228    <none>           9200/TCP         103m
    elasticsearch-es-transport       ClusterIP      None           <none>           9300/TCP         103m
    kibana-kb-http                   LoadBalancer   10.0.196.10    57.155.125.100   5601:30700/TCP   45m
    ```

2. Get password for elastic
    ```
    kubectl get secret elasticsearch-es-elastic-user -n $NAMESPACE -o go-template='{{.data.elastic | base64decode}}'
    ```

3. Navigate to Kibana > Management > Dev Tools

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
