# How to setup logging on Kubernetes using EFK (Elasticsearch, Fluentd and Kibana)

## Create a namespace for all the resources needed
Create a namespace named 'logging' or any name of choice:

kubectl create namespace logging

## Install Elasticsearch:  
cd into elasticsearch directory  

Setup the ElasticSearch master node:  

kubectl apply  -f elasticsearch-master-configmap.yaml \  
-f elasticsearch-master-service.yaml \  
-f elasticsearch-master-deployment.yaml  

Setup the ElasticSearch data node:  

kubectl apply -f elasticsearch-data-configmap.yaml \  
-f elasticsearch-data-service.yaml \  
-f elasticsearch-data-statefulset.yaml  

Setup the ElasticSearch client node:  
kubectl apply  -f elasticsearch-client-configmap.yaml \  
-f elasticsearch-client-service.yaml \   
-f elasticsearch-client-deployment.yaml


### Generate a password and store in a k8s secret:

We have enabled the xpack security module to secure the cluster, now execute the command to initialize the passwords: bin/elasticsearch-setup-passwords within the client node container (any node would work) to generate default users and passwords.  

To manually create passwords for all the Elasticsearch accounts (including account 'elastic'), run this command in the shell:  
kubectl exec -it $(kubectl get pods -n logging | grep elasticsearch-client | sed -n 1p | awk '{print $1}') -n logging -- bin/elasticsearch-setup-passwords interactive

To autogenerate passwords for all the Elasticsearch accounts (including account 'elastic'), run this command in the shell:  
kubectl exec -it $(kubectl get pods -n logging | grep elasticsearch-client | sed -n 1p | awk '{print $1}') -n logging -- bin/elasticsearch-setup-passwords auto -b  

Note the elastic user password and add it into a k8s secret like this:  

kubectl create secret generic elasticsearch-pw-elastic -n logging --from-literal password=ArKsypD2Z2isKLz52wPe

## Install Kibana:

cd into the directory kibana and run the following command:  

$ kubectl apply  -f kibana-configmap.yaml \  
-f kibana-service.yaml \  
-f kibana-deployment.yaml

### Use Ingress to make Kibana accessible publicly 

## Install Fluentd:
I made some modifications to the official Fluentd helm chart to enable filtering of logs with a time range in Kibana and also to enable the creation of daily indices (like in ELK on VMs) so that you can delete old logs (indices) based on age.  

cd back to the kubernetes_logging directory and install the Fluentd helm chart:  

helm upgrade --install fluentd fluentd -n logging

To view sources of data for elasticsearch (e.g fluentd, fluentbit etc.), you can port-forward the elasticsearch internal service:
http://elasticsearch_service:9200/_cat/indices?v

Steps to query logs with kibana are in this [documentation](https://devopscube.com/setup-efk-stack-on-kubernetes), but in Step 3, create a new Index Patten using the pattern – “logstash-*” (as seen in http://elasticsearch_service:9200/_cat/indices?v).
Also in Step 4, select “time” in the Time Filter field name option.

You can filter logs of pods using namespace, label metadata, container metadata, pod metadata, host(node) etc.
Example of Kibana query (KQL) using pod label:

kubernetes.labels.app.keyword: "aaasapigatewayinternal" 

You can query for logs, save a query search result and share the result (which saves the search result as a csv file).
