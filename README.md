# Is it Observable?
<p align="center"><img src="/image/logo.png" width="40%" alt="Prometheus Logo" /></p>

##  What is the value of the Fluent Operator
<p align="center"><img src="/image/logo_operator.png" width="40%" alt="fluent operator Logo" /></p>
Repository containing the files for the Is it Observable episode related to the Fluent Operator 


This tutorial  will :
- deploy Fluent Operator
- Create a FluentBitconfig allowing us to collect logs from a Kubernetes cluster
- create CLusterInput, ClusterFilter, ClusterParser and several ClusterOuptut to let fluent collect kubernetes logs and forward it to fluentd
- Create a Fluentdconfig that will allow us to forward the collected logs to Dynatrace.


## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm

This tutorial will generate log stream and send them to Dynatrace.
Therefore you will need a Dynatrace Tenant to be able to follow all the instructions of this tutorial . 
If you don't have any dynatrace tenant , then [let's start a trial on Dynatrace]()

### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone Github repo
```
git clone https://github.com/isItObservable/FluentOperator.git
cd FluentOperator
```

### 4. Deploy Nginx Ingress Controller
```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
kubectl apply -f nginx/deploy.yaml
```

#### 1. Get the ip address of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* hipstershop


```
IP=$(kubectl get svc nginx-ingress-nginx-controller -n ingress-nginx -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```
#### 2. update the deployment file
```
sed -i "s,IP_TO_REPLACE,$IP," hipstershop/k8s-manifest.yaml
sed -i "s,IP_TO_REPLACE,$IP," k6/k8s-manifiest.yaml
```
### 5.Prometheus
THis tutorial will use Prometheus to collect the metrics generated by K6
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --set server.nodeSelector.node-type=observability --set prometheusOperator.nodeSelector.selector.node-type=observability  --set prometheus.nodeSelector.selector.node-type=observability --set grafana.nodeSelector.selector.node-type=observability  
```

### 6.HipsterShop
```
kubectl create ns hipster-shop
kubectl -n hipster-shop create rolebinding default-view --clusterrole=view --serviceaccount=hipster-shop:default
kubectl apply -f hipstershop/k8s-manifest.yaml -n hipster-shop
```


### 6.Deploy the fluent operator
```
helm install fluent-operator --create-namespace -n kubesphere-logging-system https://github.com/fluent/fluent-operator/releases/download/v1.0.0/fluent-operator.tgz
```

### 7. Deploy
 
Before deploying our fluentbit log stream pipeline, we need to update the deployment file by adding our cluster id
```
CLUSTERID=$(kubectl get namespace kube-system -o jsonpath='{.metadata.uid}')
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," fluent/fluentbit_deployment.yaml
```

### 8. Deploy the Dynatrace ActiveGate

#### Requirement


To be able to ingest the log stream into Dynatrace, it would be requried to modify fluent/fluentbit_deployment.yaml and the active gate deployment with
* your Dynatrace Tenant URL ( your dynatrace url would be https://<TENANTID>.live.dynatrace.com )
* A dynatrace API token having the right : Ingest Logs, Paas Tokent. 
To generate your API token you will need to click on Access Tokens ( in the left menu) Follow the instruction described in dynatrace's documentation Make sure that the scope Ingest logs and Paas Token are enabled.

#### Modify the Active Gate

##### Create a service account and cluster role for accessing the Kubernetes API.
```
kubectl apply -f dynatrace/service_account.yaml
```
Create a secret holding the environment URL and login credentials for this registry, making sure to replace.
```
export ENVIRONMENT_URL=<with your environment URL (without 'http'). Example: environment.live.dynatrace.com>
export CLUSTERID=<YOUR CLUSTER ID>
export API_TOKEN=<YOUR API TOKEN>
export ENVIRONMENT_ID=<YOUR tenant id in your environment url>
kubectl create secret docker-registry tenant-docker-registry --docker-server=${ENVIRONMENT_URL} --docker-username=${ENVIRONMENT_ID} --docker-password=${API_TOKEN} -n dynatrace
kubectl create secret generic tokens --from-literal="log-ingest=${API_TOKEN}" -n dynatrace
 ```
##### Update the activegate and fluent configuration
Update the file named fluent/fluentbit_deployment.yaml and dynatrace/activegate.yaml, by running the following command :
 ```
sed -i "s,ENVIRONMENT_ID_TO_REPLACE,$ENVIRONMENT_ID," fluent/ClusterOutput_http.yaml
sed -i "s,ENVIRONMENT_URL_TO_REPLACE,$ENVIRONMENT_URL," dynatrace/activegate.yaml
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," dynatrace/activegate.yaml
sed -i "s,API_TOKEN_TO_REPLACE,$API_TOKEN," fluent/ClusterOutput_http.yaml
sed -e "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," fluent/clusterfilter.yaml
 ```
##### Deploy the activegate
 ```
kubectl apply -f dynatrace/activegate.yaml -n dynatrace
 ```
### 9. Deploy Fluentbit 

#### Fluentbit , using the output plugin
 ```
kubectl apply -f fluent/fluentbit_deployment.yaml -n kubesphere-logging-system
kubectl apply -f fluent/clusterfilter.yaml  -n kubesphere-logging-system
 ```

#### Fluentbit , let's add the ClusterOutput http to send the log stream to dynatrace
 ```
kubectl apply -f fluent/ClusterOutput_http.yaml -n kubesphere-logging-system
 ```




