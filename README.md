# Nifi Operator

1. Deploy Cert-Manager if your Kubernetes cluster doesn't have it.

```
[root@ecs-m-01 ~]# helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories

[root@ecs-m-01 ~]# helm repo list
NAME    	URL                       
jetstack	https://charts.jetstack.io

[root@ecs-m-01 ~]# helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
NAME: cert-manager
LAST DEPLOYED: Tue Jan  7 06:06:37 2025
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None

[root@ecs-m-01 ~]# oc -n cert-manager get pods
NAME                                     READY   STATUS    RESTARTS   AGE
cert-manager-5f95b55fc4-jst99            1/1     Running   0          2m15s
cert-manager-cainjector-5fd74756-j9lb6   1/1     Running   0          2m15s
cert-manager-webhook-6dc9c495d5-rr6qs    1/1     Running   0          2m15s
```

2. Download cfmctl and deploy CFM operator in the default namespace `cfm-operator-system`.

```
# ./cfmctl install --license /root/license.txt --image-repository "container.repository.cloudera.com/cloudera/cfm-operator" --image-tag "2.8.0-b94"
```

```
# kubectl create secret docker-registry cfm-credential --docker-server container.repository.cloudera.com --docker-username xxx --docker-password yyy --namespace cfm-operator-system
secret/cfm-credential created

# kubectl create secret generic cfm-operator-license --from-file=license.txt=/license.txt -n cfm-operator-system
secret/cfm-operator-license created

# kubectl -n cfm-operator-system get secret
NAME                                 TYPE                             DATA   AGE
cfm-credential                       kubernetes.io/dockerconfigjson   1      2m17s
cfm-license                          Opaque                           1      3m35s
cfm-operator-license                 Opaque                           1      61s
sh.helm.release.v1.cfm-operator.v1   helm.sh/release.v1               1      3m34s
webhook-server-cert                  kubernetes.io/tls                3      3m33s
```

```
# kubectl get crds | grep nifi
nifiregistries.cfm.cloudera.com                       2025-01-07T06:31:33Z
nifis.cfm.cloudera.com                                2025-01-07T06:31:33Z
```

```
# kubectl get pods -n cfm-operator-system
NAME                            READY   STATUS    RESTARTS   AGE
cfm-operator-5d4fbd8b98-z86vp   2/2     Running   0          10m
```

```
# kubectl create namespace dlee-nifi
namespace/dlee-nifi created

# kubectl create secret docker-registry cfm-credential --docker-server container.repository.cloudera.com --docker-username xxx --docker-password yyy --namespace dlee-nifi
secret/cfm-credential created
```

```
# kubectl -n dlee-nifi apply -f nificr-noTLS.yml
nifi.cfm.cloudera.com/mynifi created
```

```
# oc -n dlee-nifi    get all
NAME           READY   STATUS    RESTARTS   AGE
pod/mynifi-0   7/7     Running   0          3m

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/mynifi       ClusterIP   None            <none>        6007/TCP,5000/TCP   3m
service/mynifi-web   ClusterIP   10.43.121.141   <none>        8443/TCP            3m

NAME                      READY   AGE
statefulset.apps/mynifi   1/1     3d
```

```
# oc -n dlee-nifi get pvc
NAME                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
content-repository-mynifi-0      Bound    pvc-6885b555-5c9b-445b-822f-8b8e80e7becb   1Gi        RWO            longhorn       3m
data-mynifi-0                    Bound    pvc-77078d1d-9258-4d16-9d8e-dcb3f2f56b83   1Gi        RWO            longhorn       3m
flowfile-repository-mynifi-0     Bound    pvc-e248b84b-4f2c-4ac9-b3ee-d3aecc492c54   1Gi        RWO            longhorn       3m
provenance-repository-mynifi-0   Bound    pvc-baf6d48e-6725-47cb-87a1-d7c75214c06d   2Gi        RWO            longhorn       3m
state-mynifi-0                   Bound    pvc-d8457c49-fbfa-47aa-8dbc-83102d9ceca7   1Gi        RWO            longhorn       3m
```

```
# oc -n dlee-nifi get ingress
NAME         CLASS    HOSTS                             ADDRESS         PORTS   AGE
mynifi-web   <none>   mynifi.apps.dlee1.cldr.example    10.129.83.133   80      3d
```



## Case Study

<img width="1227" alt="image" src="https://github.com/user-attachments/assets/cae70a59-158c-4276-9da9-ba4a192e8467" />

<img width="786" alt="image" src="https://github.com/user-attachments/assets/1de6d52b-aa61-496d-ac76-220dd9cd4381" />
<img width="783" alt="image" src="https://github.com/user-attachments/assets/32e5137a-6a00-4f98-877b-3f4639243d24" />

<img width="1157" alt="image" src="https://github.com/user-attachments/assets/7ce0bace-e8ac-454f-92ec-94018186e7f1" />

```
$ /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --group cgroup-3

Consumer group 'cgroup-3' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
cgroup-3        ktopic-3        1          0               333             333             -               -               -
cgroup-3        ktopic-3        0          0               333             333             -               -               -
cgroup-3        ktopic-3        2          0               333             333             -               -               -
```



<img width="1381" alt="image" src="https://github.com/user-attachments/assets/f83a8aa8-1faa-4867-9080-f174897c49a6" />

<img width="1384" alt="image" src="https://github.com/user-attachments/assets/e18939c1-eb9a-4e45-816b-996307ff728b" />


<img width="1435" alt="image" src="https://github.com/user-attachments/assets/23a2b787-f188-4fc4-bce5-b8e7ea708478" />


<img width="1381" alt="image" src="https://github.com/user-attachments/assets/bfa8919f-aceb-47b0-8dc6-95770d186e69" />

<img width="1434" alt="image" src="https://github.com/user-attachments/assets/417f7170-6a08-4fb1-85d3-665952f73871" />




```
# ls -l /tmp/ktopic-3_output | wc -l
1000

# ls /tmp/ktopic-3_output
00016e83-b763-41bf-8286-a977be7d2061  3fff7416-cdb5-4005-88d3-3e3096a9ac3c  83e21cc8-b427-40b0-bf82-cb3cc68d9676  c392a563-7673-42c2-a76a-e6de74f24d0f
004587a8-d5b4-4923-869e-6ed5824f4a96  406ab935-9972-40e0-b7ad-2d96886e61af  83e71974-1025-4f69-bf51-5cab470b88ef  c3b05769-2d5b-4cb7-8b10-547fb7a84699
00895ee5-4ad7-442a-88d9-3e9866ebf952  408a50d5-439b-47f9-9a2e-473ad1d08163  83fc57f5-05b7-4c34-9a83-cfa3f01a8ae4  c3cd4ad8-30c5-4b42-a810-a4096e273baa
009255eb-8d33-4b97-9b83-1dbc70e90016  40c1f26b-9648-4ede-bcbb-0ed59b624573  84127493-9eef-4b0e-aa56-d4ae89c46185  c3e10c55-c960-4106-8a90-0a8cf86f510f
.....

# more 2d8dbfab-7302-46f1-9e07-36f14b3bcc83
{"message": "Message number 154"}
```


