# prom-thanos

1. Create S3 Bucket (I created prom-thanos-store)
2. Add prometheus-community repo for Prometheus.
3. Create secret with s3 bucket configuration
4. Install prometheus with thanos sidecar.
5. Install Thanos Querier service and deployment
6. Create Thanos store


**1. Create S3 Bucket (I created prom-thanos-store)**

aws s3api create-bucket --bucket prom-thanos-store --region us-east-2 --create-bucket-configuration LocationConstraint=us-east-2

**2. Add prometheus-community repo for Prometheus.**

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    help repo list
    helm repo update

**3. Create secret with s3 bucket configuration**

    In uitilities namespace we will creete the Prometheus stack.

    kubectl create ns utilities

    Put your AWS account user access key and secret access, so that thanos sidecar can connect push the metrics scrape by prometheus to S3 bucket.

    kubectl -n utilities create secret generic thanos-objstore-config –from-file=thanos.yaml=thanos-storage-config.yaml

**4. Install prometheus with thanos sidecar.**

**These are the changes which we need to add to add thanos sidecar to the prometheus.**

    thanos:         # add Thanos Sidecar
      tag: v0.13.0   # a specific version of Thanos
      objectStorageConfig: # blob storage configuration to upload metrics 
        key: thanos.yaml
        name: thanos-objstore-config


    export prom_version=10.0.0

    helm upgrade --install \
    prom    \
    prometheus-community/kube-prometheus-stack   \
    --version "${prom_version}"    \
    --namespace utilities   \
    --set nameOverride=prometheus-operator \
    -f prometheus-operator-values.yaml

    kubectl apply -f thanos-sidecar-svc.yaml -n utilities
    
    # kubectl get pods -n utilities 
    NAME                                                   READY   STATUS    RESTARTS   AGE
    alertmanager-prom-prometheus-operator-alertmanager-0   2/2     Running   0          145m
    prom-grafana-6b8d7c75fd-rd2qj                          2/2     Running   0          24s
    prom-kube-state-metrics-789fd98679-7wvw4               1/1     Running   0          145m
    prom-prometheus-node-exporter-9rw96                    1/1     Running   0          145m
    prom-prometheus-node-exporter-jv6gb                    1/1     Running   0          145m
    prom-prometheus-node-exporter-xz42l                    1/1     Running   0          145m
    prom-prometheus-operator-operator-7f576586cd-9ldmq     2/2     Running   0          145m
    **prometheus-prom-prometheus-operator-prometheus-0       4/4     Running   1          18s**
    
    # kubectl get svc -n utilities
    NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
    alertmanager-operated                   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   145m
    prom-grafana                            ClusterIP   10.100.67.94     <none>        80/TCP                       145m
    prom-kube-state-metrics                 ClusterIP   10.100.140.243   <none>        8080/TCP                     145m
    prom-prometheus-node-exporter           ClusterIP   10.100.120.239   <none>        9100/TCP                     145m
    prom-prometheus-operator-alertmanager   ClusterIP   10.100.125.47    <none>        9093/TCP                     145m
    prom-prometheus-operator-operator       ClusterIP   10.100.220.23    <none>        8080/TCP,443/TCP             145m
    prom-prometheus-operator-prometheus     ClusterIP   10.100.94.245    <none>        9090/TCP                     145m
    prometheus-operated                     ClusterIP   None             <none>        9090/TCP,10901/TCP           145m
    
    Now edit below services to set type as LoadBalancer, so that we can access it globally.
    
    # kubectl edit svc prom-grafana -n utilities
    service/prom-grafana edited

    # kubectl edit svc prom-prometheus-operator-prometheus -n utilities
    service/prom-prometheus-operator-prometheus edited
    
      NAME                                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                           AGE
    alertmanager-operated                   ClusterIP      None             <none>                                                                    9093/TCP,9094/TCP,9094/UDP        8h
    prom-grafana                            LoadBalancer   10.100.67.94     a4a649d0c29444dbd96134af449a3673-1178011107.us-east-2.elb.amazonaws.com   80:30283/TCP                      8h
    prom-kube-state-metrics                 ClusterIP      10.100.140.243   <none>                                                                    8080/TCP                          8h
    prom-prometheus-node-exporter           ClusterIP      10.100.120.239   <none>                                                                    9100/TCP                          8h
    prom-prometheus-operator-alertmanager   ClusterIP      10.100.125.47    <none>                                                                    9093/TCP                          8h
    prom-prometheus-operator-operator       ClusterIP      10.100.220.23    <none>                                                                    8080/TCP,443/TCP                  8h
    prom-prometheus-operator-prometheus     LoadBalancer   10.100.94.245    ab2187a04566e4ea4b42a58cb6446a80-1691844893.us-east-2.elb.amazonaws.com   9090:30818/TCP                    8h
    prometheus-operated                     ClusterIP      None             <none>                                                                    9090/TCP,10901/TCP                8h
    
**5. Install Thanos Querier service and deployment**

    kubectl apply -f querier-service.yaml 
    kubectl apply -f querier-deployment.yaml
    kubectl apply -f thanos-store.yaml 


    # kubectl get pods -n utilities
    NAME                                                   READY   STATUS    RESTARTS   AGE
    alertmanager-prom-prometheus-operator-alertmanager-0   2/2     Running   0          8h
    prom-grafana-6b8d7c75fd-rd2qj                          2/2     Running   0          5h48m
    prom-kube-state-metrics-789fd98679-7wvw4               1/1     Running   0          8h
    prom-prometheus-node-exporter-9rw96                    1/1     Running   0          8h
    prom-prometheus-node-exporter-jv6gb                    1/1     Running   0          8h
    prom-prometheus-node-exporter-xz42l                    1/1     Running   0          8h
    prom-prometheus-operator-operator-7f576586cd-9ldmq     2/2     Running   0          8h
    prometheus-prom-prometheus-operator-prometheus-0       4/4     Running   1          4h11m
    thanos-querier-598dcf4f54-ztf7p                        1/1     Running   0          5h42m
    thanos-store-0                                         1/1     Running   0          5h39m

    # kubectl edit svc prom-grafana -n utilities
    service/prom-grafana edited

    # kubectl edit svc prom-prometheus-operator-prometheus -n utilities
    service/prom-prometheus-operator-prometheus edited


     kubectl get svc -n utilities

    NAME                                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                           AGE
    alertmanager-operated                   ClusterIP      None             <none>                                                                    9093/TCP,9094/TCP,9094/UDP        8h
    prom-grafana                            LoadBalancer   10.100.67.94     a4a649d0c29444dbd96134af449a3673-1178011107.us-east-2.elb.amazonaws.com   80:30283/TCP                      8h
    prom-kube-state-metrics                 ClusterIP      10.100.140.243   <none>                                                                    8080/TCP                          8h
    prom-prometheus-node-exporter           ClusterIP      10.100.120.239   <none>                                                                    9100/TCP                          8h
    prom-prometheus-operator-alertmanager   ClusterIP      10.100.125.47    <none>                                                                    9093/TCP                          8h
    prom-prometheus-operator-operator       ClusterIP      10.100.220.23    <none>                                                                    8080/TCP,443/TCP                  8h
    prom-prometheus-operator-prometheus     LoadBalancer   10.100.94.245    ab2187a04566e4ea4b42a58cb6446a80-1691844893.us-east-2.elb.amazonaws.com   9090:30818/TCP                    8h
    prometheus-operated                     ClusterIP      None             <none>                                                                    9090/TCP,10901/TCP                8h
    thanos-querier                          LoadBalancer   10.100.65.182    acdca23129c2443f6a7028c3878cd65f-699227861.us-east-2.elb.amazonaws.com    10901:31988/TCP,9090:30721/TCP    5h46m
    thanos-sidecar                          LoadBalancer   10.100.82.94     ad3f6320d931143bb8f7edcb2f528851-16449826.us-east-2.elb.amazonaws.com     10901:30424/TCP                   5h48m
    thanos-store                            LoadBalancer   10.100.8.89      a4b74d80a6fd94cedb5c6980c4d96017-1850067811.us-east-2.elb.amazonaws.com   10901:30178/TCP,10902:30621/TCP   5h23m


      Thanos querier dashboard:
  
  <img width="1529" alt="image" src="https://user-images.githubusercontent.com/74225291/163675401-5f70fd5f-b649-418f-943f-d34f3f6e5dcc.png">


      Grafana Dashborad: 
      We can add new Datasource for Thanos Querier in Grafana.
  
  <img width="1318" alt="image" src="https://user-images.githubusercontent.com/74225291/163675473-aa7592a8-78c6-4b0f-ad3e-c1d0077819b2.png">
  
  
      S3 bucket data:
  
 <img width="1217" alt="image" src="https://user-images.githubusercontent.com/74225291/163675531-566cf351-d3e4-4ed7-9906-d595b0e3e2c9.png">


