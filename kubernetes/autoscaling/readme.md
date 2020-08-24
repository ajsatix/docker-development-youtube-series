# Kubernetes Autoscaling Guide

## Cluster Autoscaling

Cluster autoscaler allows us to scale cluster nodes when they become full <br/>

## Horizontal Pod Autoscaling

HPA allows us to scale pods when their resource utilisation goes over a threshold <br/>

## Requirements

### A Cluster 

* For both autoscaling guides, we'll need a cluster. <br/>
* For `Cluster Autoscaler` You need a cloud based cluster that supports the cluster autoscaler <br/>
* For `HPA` We'll use [kind](http://kind.sigs.k8s.io/)

### Cluster Autoscaling - Creating an AKS Cluster

```
# azure example

NAME=aks-getting-started
RESOURCEGROUP=aks-getting-started
SERVICE_PRINCIPAL=
SERVICE_PRINCIPAL_SECRET=

az aks create -n $NAME \
--resource-group $RESOURCEGROUP \
--location australiaeast \
--kubernetes-version 1.16.10 \
--nodepool-name default \
--node-count 1 \
--node-vm-size Standard_F4s_v2  \
--node-osdisk-size 250 \
--service-principal $SERVICE_PRINCIPAL \
--client-secret $SERVICE_PRINCIPAL_SECRET \
--output none \
--enable-cluster-autoscaler \
--min-count 1 \
--max-count 5
```

### Horizontal Pod Autocaling - Creating a Kind Cluster

My Node has 6 CPU cores for this demo <br/>

```
kind create cluster --name hpa --image kindest/node:v1.18.4
```


### Metric Server

* For `Cluster Autoscaler` - On cloud-based clusters, Metric server may already be installed. <br/>
* For `HPA` - We're using kind

[Metric Server](https://github.com/kubernetes-sigs/metrics-server) provides container resource metrics for use in autoscaling pipelines <br/>

Because I run K8s `1.18` in `kind`, the Metric Server version i need is `0.3.7` <br/>
We will need to deploy Metric Server [0.3.7](https://github.com/kubernetes-sigs/metrics-server/releases/tag/v0.3.7) <br/>
I used `components.yaml`from the release page link above. <br/>

<b>Important Note</b> : For Demo clusters (like `kind`), you will need to disable TLS <br/>
You can disable TLS by adding the following to the metrics-server container args <br/>

<b>For production, make sure you remove the following :</b> <br/>

```
- --kubelet-insecure-tls
- --kubelet-preferred-address-types="InternalIP"

```

Deployment: <br/>


```
cd kubernetes\autoscaling
kubectl -n kube-system apply -f .\metric-server\metricserver-0.3.7.yaml

#test 
kubectl -n kube-system get pods

#wait for metrics to populate
kubectl top nodes

```

## Example Application

For all autoscaling guides, we'll need a simple app, that generates some CPU load <br/>

* Build the app
* Push it to a registry
* Ensure resource requirements are set
* Deploy it to Kubernetes
* Ensure metrics are visible for the app

```
# build

cd kubernetes\autoscaling\application-cpu
docker build . -t aimvector/application-cpu:v1.0.0

# push
docker push aimvector/application-cpu:v1.0.0

# resource requirements
resources:
  requests:
    memory: "50Mi"
    cpu: "500m"
  limits:
    memory: "500Mi"
    cpu: "2000m"

# deploy 
kubectl apply -f deployment.yaml

# metrics
kubectl top pods

```

## Generate some traffic


```
# get a terminal to the traffic-generator
kubectl exec -it traffic-generator sh
# install wrk
apk add --no-cache wrk curl

# simulate some load
wrk -c 5 -t 5 -d 99999 -H "Connection: Close" http://application-cpu

kubectl scale deploy/application-cpu --replicas 2
```