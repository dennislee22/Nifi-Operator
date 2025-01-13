# Nifi Operator: Visual Data Flow Management

<img width="661" alt="image" src="https://github.com/user-attachments/assets/3e3d055f-021a-4d39-8f05-9aa70069e6a0" />

Apache NiFi is a powerful data integration platform designed to automate the movement, transformation, and management of data across diverse systems in real time. Built on a flow-based programming paradigm, NiFi enables users to design dataflows by linking processors that perform specific functions such as data ingestion, transformation, routing, and delivery. These flows are visually designed via NiFi’s intuitive web interface, which simplifies the creation and monitoring of complex workflows without requiring extensive coding knowledge. In this article, I will walk you through the installation process of the CFM (Cloudera Flow Management) operator on a Kubernetes cluster, leveraging the cloud-native advantages that Kubernetes offers, such as self-healing, automated rollouts, and scalability.

## Deployment Steps
1. Deploy `Cert-Manager` if your Kubernetes cluster doesn't have it.

```
# helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories

# helm repo list
NAME    	URL                       
jetstack	https://charts.jetstack.io

# helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
NAME: cert-manager
LAST DEPLOYED: Tue Jan  7 06:06:37 2025
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None

# oc -n cert-manager get pods
NAME                                     READY   STATUS    RESTARTS   AGE
cert-manager-5f95b55fc4-jst99            1/1     Running   0          2m15s
cert-manager-cainjector-5fd74756-j9lb6   1/1     Running   0          2m15s
cert-manager-webhook-6dc9c495d5-rr6qs    1/1     Running   0          2m15s
```

2. Download [cfmctl](https://docs.cloudera.com/cfm-operator/2.9.0/installation/topics/cfm-op-install-overview.html) and deploy CFM operator in the namespace `cfm-operator-system`.

```
# ./cfmctl install --license /root/license.txt --image-repository "container.repository.cloudera.com/cloudera/cfm-operator" --image-tag "2.8.0-b94"
```

3. Create a secret object in the same namespace to store your Cloudera credentials.
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

4. Ensure Nifi CRDs are installed.
```
# kubectl get crds | grep nifi
nifiregistries.cfm.cloudera.com                       2025-01-07T06:31:33Z
nifis.cfm.cloudera.com                                2025-01-07T06:31:33Z
```

5. Ensure CFM pod is up and `Running`. 
```
# kubectl get pods -n cfm-operator-system
NAME                            READY   STATUS    RESTARTS   AGE
cfm-operator-5d4fbd8b98-z86vp   2/2     Running   0          10m
```

6. Create another namespace to host the Nifi cluster. Create a secret object in the same namespace to store your Cloudera credentials.
```
# kubectl create namespace dlee-nifi
namespace/dlee-nifi created

# kubectl create secret docker-registry cfm-credential --docker-server container.repository.cloudera.com --docker-username xxx --docker-password yyy --namespace dlee-nifi
secret/cfm-credential created
```

7. Create a Nifi cluster by applying `nificr-noTLS.yml` file. There's only 1 Nifi replica pod in this cluster. You may spin up more replicas in the actual production environment.
```
# kubectl -n dlee-nifi apply -f nificr-noTLS.yml
nifi.cfm.cloudera.com/mynifi created
```

8. Ensure the Nifi pod is up and `Running`. 
```
# kubectl -n dlee-nifi    get all
NAME           READY   STATUS    RESTARTS   AGE
pod/mynifi-0   7/7     Running   0          3m

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/mynifi       ClusterIP   None            <none>        6007/TCP,5000/TCP   3m
service/mynifi-web   ClusterIP   10.43.121.141   <none>        8080/TCP            3m

NAME                      READY   AGE
statefulset.apps/mynifi   1/1     3d
```

9. The deployment file would also create the following PVC automatically.
```
# kubectl -n dlee-nifi get pvc
NAME                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
content-repository-mynifi-0      Bound    pvc-6885b555-5c9b-445b-822f-8b8e80e7becb   1Gi        RWO            longhorn       3m
data-mynifi-0                    Bound    pvc-77078d1d-9258-4d16-9d8e-dcb3f2f56b83   1Gi        RWO            longhorn       3m
flowfile-repository-mynifi-0     Bound    pvc-e248b84b-4f2c-4ac9-b3ee-d3aecc492c54   1Gi        RWO            longhorn       3m
provenance-repository-mynifi-0   Bound    pvc-baf6d48e-6725-47cb-87a1-d7c75214c06d   2Gi        RWO            longhorn       3m
state-mynifi-0                   Bound    pvc-d8457c49-fbfa-47aa-8dbc-83102d9ceca7   1Gi        RWO            longhorn       3m
```

10. You may now browse the Nifi URL `mynifi.apps.dlee1.cldr.example` as exposed to the external network via ingress as shown below.
```
# kubectl -n dlee-nifi get ingress
NAME         CLASS    HOSTS                             ADDRESS         PORTS   AGE
mynifi-web   <none>   mynifi.apps.dlee1.cldr.example    10.129.83.133   80      3d
```


## Case Study

Here’s an illustration of a NiFi use case: ingesting data from a Kafka topic and sinking it into a remote Linux box via SSH. The flow begins with the `ConsumeKafka` processor, which pulls messages from the Kafka topic. Once ingested, the data can be optionally processed or transformed using processors like UpdateAttribute or ExecuteScript. After any necessary transformations, the `PutSFTP` processor is used to send the data to a specified directory on the remote Linux server via SSH. NiFi’s built-in features, including error handling, backpressure management, and provenance tracking, ensure data integrity and flow reliability throughout the entire process, enabling seamless, real-time data movement from Kafka to a remote server.

1. Create `ConsumeKafka` and `PutSFTP` processors and link them together. Both processors are in `Stop` mode.
<img width="1227" alt="image" src="https://github.com/user-attachments/assets/cae70a59-158c-4276-9da9-ba4a192e8467" />

2. Configure `ConsumeKafka` as follows.
<img width="786" alt="image" src="https://github.com/user-attachments/assets/1de6d52b-aa61-496d-ac76-220dd9cd4381" />

3. Configure `PutSFTP` as follows.
<img width="783" alt="image" src="https://github.com/user-attachments/assets/32e5137a-6a00-4f98-877b-3f4639243d24" />

4. On an external Jupyter Notebook, run a simple Python code to create messages into a Kafka topic.
<img width="1157" alt="image" src="https://github.com/user-attachments/assets/7ce0bace-e8ac-454f-92ec-94018186e7f1" />

5. Upon completion, check the LAG status of consumer group `cgroup-3`.
```
$ /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --group cgroup-3

Consumer group 'cgroup-3' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
cgroup-3        ktopic-3        1          0               333             333             -               -               -
cgroup-3        ktopic-3        0          0               333             333             -               -               -
cgroup-3        ktopic-3        2          0               333             333             -               -               -
```

6. Run the `ConsumeKafka` processor but not the `PutSFTP` processor.
<img width="1435" alt="image" src="https://github.com/user-attachments/assets/23a2b787-f188-4fc4-bce5-b8e7ea708478" />

7. Right click the `ConsumeKafka` processor and select the "View Data Provenance" button. Note that the processor has successfully ingested the 1000 messages from the Kafka cluster.
<img width="1384" alt="image" src="https://github.com/user-attachments/assets/e18939c1-eb9a-4e45-816b-996307ff728b" />

8. Right click the `Success` connection and select the "List queue" button. Note that queue has been filled up with 1000, pending for the next data flow action.
<img width="1381" alt="image" src="https://github.com/user-attachments/assets/bfa8919f-aceb-47b0-8dc6-95770d186e69" />

9. Next, run the `PutSFTP` processor. Note that the queue has been processed to the next flow hop and emptied.
<img width="1434" alt="image" src="https://github.com/user-attachments/assets/417f7170-6a08-4fb1-85d3-665952f73871" />

10. Finally, check the destination (Linux box) and ensure that all 1000 messages have successfully been ingested into the configured directory.

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


