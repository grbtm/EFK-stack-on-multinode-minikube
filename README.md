# EFK-stack-on-multinode-minikube
## What
Basic logging example with a EFK stack on a multinode (Docker based) minikube cluster.
Based on the [Digital ocean tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-elasticsearch-fluentd-and-kibana-efk-logging-stack-on-kubernetes) and adapted for a multinode minikube cluster.

## Setup overview
The ElasticSearch cluster is defined as a StatefulSet, Kibana as Deployment and Service and additionally we'll have a simple counter app in a Pod.
There are two different approaches on how to handle logging with fluentd:

### Cluster-level logging with Fluentd as Daemonset
Fluentd is defined as a DaemonSet (thus the requirement of this setup to have role-based access control enabled) with a logging agent on every node which fetches container stdout and stderr and writes it to a single Elastic index.

### Application logging with Fluentd as sidecar
Logs are separated per application and written to app-specific Elastic indices. Each app Pod therefore needs to come with a dedicated fluentd sidecar container which fetches the log file from a shared mounted dir and writes it to the Elastic index.

## Usage
```bash
minikube start --vm-driver=docker --nodes 4 -p logging-sandbox
```
Make sure that kubectl has the correct context `logging-sandbox`:
```bash
kubectl config current-context
```
Create a namespace for EFK:
```bash
kubectl create -f kube-logging.yaml
```

Create Elastic Service:
```bash
kubectl create -f elasticsearch_svc.yaml
```
[Workaround](https://github.com/kubernetes/minikube/issues/12165#issuecomment-1052642443) for multinode minikube cluster due to [issue](https://github.com/kubernetes/minikube/issues/12165#issuecomment-1052642443) with PersistentVolumeClaims of StatefulSet
```bash
curl https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml | sed 's/\/opt\/local-path-provisioner/\/var\/opt\/local-path-provisioner/ ' | kubectl apply -f -
  kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
  kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Create Elastic StatefulSet:
```bash
kubectl create -f elasticsearch_statefulset.yaml
```

Create Kibana Deployment and Service:
```bash
kubectl create -f kibana.yaml
```

Once it's rolled out, get Kibana pod name and forward local port to pod to access web UI:
```bash
kubectl port-forward kibana-pod-name 5601:5601 --namespace=kube-logging
```

### For cluster wide logging setup with Fluentd DaemonSet:

Create Fluentd DaemonSet:
```bash
kubectl create -f fluentd-deamonset.yaml
```
Open Kibana at http://localhost:5601 and create new index pattern with `logstash-*` wildcard pattern, time filter by `@timestamp` field and
all the logs of the cluster should be available through the Discover UI.

### For separated app logging with Fluentd sidecar container:
Create Fluentd ConfigMap:
```bash
kubectl create -f fluentd-sidecar-configmap-counter-app.yaml
```

Deploy the simple counter app along with fluentd sidecar container with dedicated logging agent: 
```bash
kubectl create -f counter-app-w-fluentd-sidecar.yaml
```
Open Kibana at http://localhost:5601 and create new index pattern named after your `DOMAIN_UID` env var in counter-app-w-fluentd-sidecar.yaml
