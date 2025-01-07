# nifi-operator


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

# kubectl create secret generic cfm-operator-license --from-file=license.txt=/root/dennis_lee_2024_2025_cloudera_license.txt -n cfm-operator-system
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
apiVersion: cfm.cloudera.com/v1alpha1
kind: Nifi
metadata:
  name: mynifi
spec:
  replicas: 3
  image:
    repository: container.repository.cloudera.com/cloudera/cfm-nifi-k8s
    tag: 2.9.0-b96-nifi_1.27.0.2.3.14.0-14
    pullSecret: docker-pull-secret
  tiniImage:
    repository: container.repository.cloudera.com/cloudera/cfm-tini
    tag: 2.9.0-b96
    pullSecret: docker-pull-secret
```

```
# kubectl -n dlee-nifi apply -f nificr-1.yaml 
nifi.cfm.cloudera.com/mynifi created
```

```
```
