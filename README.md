# How to setup logging on Kubernetes using EFK (Elasticsearch, Fluentd and Kibana)

## Create a namespace for all the resources needed
Create a namespace named 'logging' or any name of choice:

kubectl create namespace logging

## Install Elasticsearch (with X-pack security):  
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

NOTE: The above k8s deployment and statefulset files are for a single-node cluster. If you deploy this in a multiple-node cluster the master, data, and client pods may end up in different nodes and they will not be able to communicate effectively. Errors such as 'master_not_discovered_exception' may show up.

For multiple-node cluster, use the files suffixed with "-node-affinity". Node affinity will force the deployments and statefulset to be deployed in a particular node (as seen in these k8s manifests) and Priority Class will prevent eviction of pods. Look for a label that is unique to a particular reliable node you want Elasticsearch installed in, and replace them with what is in the "-node-affinity" suffixed k8s files.

### Generate a password and store in a k8s secret:

We have enabled the xpack security module to secure the cluster, now execute the command to initialize the passwords: bin/elasticsearch-setup-passwords within the data node container (any node would work, but I prefer data node, so that the passwords setup will save in its PV?) to generate default users and passwords.  

To manually create passwords for all the Elasticsearch accounts (including account 'elastic'), run this command in the shell:  
kubectl exec -it $(kubectl get pods -n logging | grep elasticsearch-data | sed -n 1p | awk '{print $1}') -n logging -- bin/elasticsearch-setup-passwords interactive

To autogenerate passwords for all the Elasticsearch accounts (including account 'elastic'), run this command in the shell:  
kubectl exec -it $(kubectl get pods -n logging | grep elasticsearch-data | sed -n 1p | awk '{print $1}') -n logging -- bin/elasticsearch-setup-passwords auto -b  

Note the elastic user password and add it into a k8s secret like this:  

kubectl create secret generic elasticsearch-pw-elastic -n logging --from-literal password=<secret_value_not_base64_value>

## Install Kibana:

cd into the directory kibana and run the following command:  

$ kubectl apply  -f kibana-configmap.yaml \  
-f kibana-service.yaml \  
-f kibana-deployment.yaml

### Use Ingress to make Kibana accessible publicly 

## Install Fluentd:
I made some modifications to the official Fluentd helm chart to enable filtering of logs with a time range in Kibana and also to enable the creation of daily indices (like in ELK on VMs) so that you can delete old logs (indices) based on age. 

NOTE: For k8s clusters of v1.25+, I changed "podSecurityPolicy: enabled" to "true" from "false" in the values.yaml file. This is because "PodSecurityPolicy - policy/v1beta1" resource was removed from 1.25 upwards. Please chech the [official Fluentd helm chart](https://github.com/fluent/helm-charts/tree/e36eec9eb85bf875e178eeb51f19170ad58216c2/charts/fluentd) for new updates.

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


## Install Curator:  
Curator is installed to help automatically delete old Fluentd logs (indices) based on age, using a cronjob. The curator directory is a modified helm chart of the official elasticsearch-curator chart on ArtifactHUB. The modifications were made on the values.yaml file to connect to our elasticsearch stack and delete Fluentd indices (logstash-*_) that are over 7 days old. This is done to manage storage as Fluentd generates logs of logs daily, depending on the amount of pods running in the cluster.

Because the daily jobs to delete old indices will not be removed automatically after it runs, we create a new namespace so that the pods that each daily run creates will not be mixed up with our other pods.

kubectl create namespace curator

Install the Curator helm chart:  

helm upgrade --install curator curator -n curator  OR
helm upgrade --install curator ./curator -n curator 

The cron job will run as per set in the values.yaml file of the chart, deleting the old Fluentd indices.
