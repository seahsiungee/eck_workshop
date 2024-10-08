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

# Useful Commands
```
# Watch events related to statefulset
kubectl events -n $NAMESPACE --for=StatefulSet/elasticsearch-es-default
kubectl events -n $NAMESPACE --for=StatefulSet/elastic-operator
```

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

Check that statefulset & pods are updated
```
kubectl get sts,pods -A --field-selector metadata.namespace=$NAMESPACE
```

You should see 
```bash
azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl get sts,pods -A --field-selector metadata.namespace=$NAMESPACE
NAMESPACE   NAME                                       READY   AGE
siu         statefulset.apps/elastic-operator          1/1     3h19m
siu         statefulset.apps/elasticsearch-es-frozen   1/1     5m14s
siu         statefulset.apps/elasticsearch-es-hot      2/2     5m14s
siu         statefulset.apps/elasticsearch-es-master   3/3     5m14s

NAMESPACE   NAME                             READY   STATUS    RESTARTS   AGE
siu         pod/elastic-operator-0           1/1     Running   0          3h19m
siu         pod/elasticsearch-es-frozen-0    1/1     Running   0          5m14s
siu         pod/elasticsearch-es-hot-0       1/1     Running   0          5m14s
siu         pod/elasticsearch-es-hot-1       1/1     Running   0          5m14s
siu         pod/elasticsearch-es-master-0    1/1     Running   0          5m14s
siu         pod/elasticsearch-es-master-1    1/1     Running   0          4m13s
siu         pod/elasticsearch-es-master-2    1/1     Running   0          3m9s
siu         pod/kibana-kb-6b6f45bcbb-krk2t   1/1     Running   0          139m
```

Verify Elasticsearch Nodes via API in Kibana > Management > Dev Tools

Execute the following command in Dev Tools
```
GET _cat/nodes?v&s=name
```

You should see a response similar to the below
```
ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.244.5.243           59          72   3    0.07    0.28     0.23 f         -      elasticsearch-es-frozen-0
10.244.3.11            56          74   4    0.17    0.38     0.26 hirst     -      elasticsearch-es-hot-0
10.244.2.81            18          75   4    0.20    0.48     0.36 hirst     -      elasticsearch-es-hot-1
10.244.4.161           21          82   3    0.08    0.29     0.25 m         -      elasticsearch-es-master-0
10.244.3.252           24          83   5    0.17    0.38     0.26 m         *      elasticsearch-es-master-1
10.244.2.47            21          81   3    0.20    0.48     0.36 m         -      elasticsearch-es-master-2
```

## 3. Update deployment to enable monitoring

Create secret for stack-mon
```
# [Optional] Get Password for elastic user of central-stack-mon
kubectl get secret central-stack-mon-es-elastic-user -n stack-mon -o go-template='{{.data.elastic | base64decode}}'

# create `central-stack-mon-es-ref` secret in your namespace
kubectl -n $NAMESPACE create secret generic central-stack-mon-es-ref --from-literal=username=elastic --from-literal=url=https://central-stack-mon-es-http.stack-mon.svc:9200 --from-file=password=/home/azureuser/eck_workshop/common/password.txt --from-file=/home/azureuser/eck_workshop/common/ca.crt
```

Apply Elasticsearch Cluster manifest with monitoring specification to `central-stack-mon-es-ref` secret
```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/elasticsearch_hot_frozen_monitored.yaml -n $NAMESPACE
```

Apply the same for Kibana
```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/kibana_monitored.yaml -n $NAMESPACE
```

## 4. Add secure setting for blob storage to keystore
Create blob storage credentials as secret 
```
kubectl create secret generic az-storage-credentials --from-file=/home/azureuser/eck_workshop/2_elastic_stack/snapshot/azure.client.default.account --from-file=/home/azureuser/eck_workshop/2_elastic_stack/snapshot/azure.client.default.key -n $NAMESPACE
```

Check that secret is created
```
kubectl get secret/az-storage-credentials -n $NAMESPACE
```

You should see
```shell
azureuser@siu-scb-wks-bastion:~/eck_workshop$ kubectl get secret/az-storage-credentials -n $NAMESPACE
NAME                     TYPE     DATA   AGE
az-storage-credentials   Opaque   2      3m14s
```

Apply elasticsearch manifest with secure settings referencing `az-storage-credentials` secret
```
kubectl apply -f /home/azureuser/eck_workshop/2_elastic_stack/elasticsearch_hot_frozen_monitored_snapshot.yaml  -n $NAMESPACE
```

### Configure repository in your Elasticsearch deployment

**Execute all commands in this section in Kibana Dev Tools**

A. Create Repository referencing client credentials added to Elasticsearch keystore.

Replace `<namespace>` with your namespace in the following command
```
PUT /_snapshot/azure_repository
{
  "type": "azure",
  "settings": {
    "container": "workshop",
    "base_path": "<namespace>",
    "client": "default"
  }
}
```

B. Verify that snapshot repository is reachable
```
POST _snapshot/azure_repository/_verify
```

You should see 
```
{
  "nodes": {
    "zz2vzXwURJGbTI1hDEMRSQ": {
      "name": "elasticsearch-es-hot-0"
    },
    "7MFXguqCQ8K1UeodLUK0JA": {
      "name": "elasticsearch-es-master-1"
    },
    "y03JiyvdQ5Slykwh4uWv9g": {
      "name": "elasticsearch-es-frozen-0"
    },
    "r6xc01aXSFG5Jb_gGVZMUQ": {
      "name": "elasticsearch-es-master-0"
    },
    "jePcg-SKQLy4hj9tpgHpbA": {
      "name": "elasticsearch-es-master-2"
    },
    "5ef2XiGGQ-aiU59JWXd7Ag": {
      "name": "elasticsearch-es-hot-1"
    }
  }
}
```

C. Take a snapshot
```
POST _snapshot/azure_repository/2024_16_09
{
  "indices": "*",
  "include_global_state": true
}
```

D. Check snapshot status
```
GET _snapshot/azure_repository/2024_16_09/_status
```
