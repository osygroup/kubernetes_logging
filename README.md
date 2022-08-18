# How to setup logging on Kubernetes using EFK (Elasticsearch, Fluentd and Kibana)

## Create a namespace for all the resources needed
Create a namespaced nammed 'logging' or any name of choice:

kubectl create namespace logging

## Install Elasticsearch (a statefulSet):

In the Kibana deployment yaml file, I swapped http://elasticsearch:9200 with http://elasticsearch.namespace.svc.cluster.local:9200 (actually this isn't necessary as Kibana and Elasticsearch are in the same namespace).
For Kibana I used internal service and ingress instead of the nodePort yaml file provided.

kubectl apply -n logging -f elasticsearch/es-svc.yaml

kubectl apply -n logging -f elasticsearch/es-sts.yaml

## Install Kibana (a deployment):

kubectl apply -n logging -f kibana/kibana-deployment.yaml

kubectl apply -n logging -f kibana/kibana-svc-internal.yaml

### Use Ingress to make Kibana accessible publicly 

## Install Fluentd (a daemonSet):
In the values.yaml file, search for 'elasticsearch-master' (this appears twice) and replace it with 'elasticsearch'. Using 'elasticsearch' as host is based on the assumption that your elasticsearch internal service is deployed in the same namespace as that you want to install Fluentd in. Else use the elasticsearch's internal service DNS url.
In the values.yaml file, also change 'keep_time_key' from 'false' to 'true' in the parser block. This block is found in the fileConfigs section of the file. If this is not done, you can't filter logs with a time range in Kibana.

helm upgrade --install fluentd fluentd -n logging

To view sources of data for elasticsearch (e.g fluentd, fluentbit etc.), you can port-forward the elasticsearch internal service:
http://elasticsearch_service:9200/_cat/indices?v

Steps to query logs with kibana are in the [Devopscube documentation](https://devopscube.com/setup-efk-stack-on-kubernetes), but in Step 3, create a new Index Patten using the pattern – “fluentd” (as seen in http://elasticsearch_service:9200/_cat/indices?v).
Also in Step 4, select “time” in the Time Filter field name option.

You can filter logs of pods using namespace, label metadata, container metadata, pod metadata, host(node) etc.
Example of Kibana query (KQL) using pod label:

kubernetes.labels.app.keyword: "aaasapigatewayinternal" 

You can query for logs, save a query search result and share the result (which saves the search result as a csv file).
