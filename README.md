# How to setup logging on Kubernetes using EFK (Elasticsearch, Fluentd and Kibana)
The Elasticsearch version used in this repo is version 8.7.0

## Create a namespace for all the resources needed
Create a namespace named 'logging' or any name of choice:

kubectl create namespace logging

## Create p12 certificate and install as secret
Elasticsearch v8 now enforces minimal security (x-pack) by default. 
If your cluster has multiple nodes, then you must configure TLS between nodes. Production mode clusters will not start if you do not enable TLS.
My elasticsearch setup has 3 nodes (master, client and data) so it requires basic security.

This [documentation](https://medium.com/@musabdogan/enabling-elasticsearch-xpack-security-on-an-unsecured-cluster-79f6ea4023dd) shows how you can create the p12 certificate with a longer expiry than the default 1095 days:
You will need to create the certificate on single elastic node and then copy it out. You can run Elasticsearch in a docker container and copy out the certificate. There is a passwordless certificate in the elasticsearch driectory that can be used.
cd into elasticsearch directory, then deploy the certificate as a secret to the cluster:

kubectl create secret generic elastic-certificates-p12 -n logging --from-file=elastic-certificates.p12

## Install Elasticsearch:  
Still in the elasticsearch directory, setup the ElasticSearch master node:  

kubectl apply -f elasticsearch-master-configmap.yaml \  
-f elasticsearch-master-service.yaml \  
-f elasticsearch-master-statefulset.yaml  

Setup the ElasticSearch data node:  

kubectl apply -f elasticsearch-data-configmap.yaml \  
-f elasticsearch-data-service.yaml \  
-f elasticsearch-data-statefulset.yaml  

Setup the ElasticSearch client node:  
kubectl apply -f elasticsearch-client-configmap.yaml \  
-f elasticsearch-client-service.yaml \   
-f elasticsearch-client-statefulset.yaml

NOTE: The above k8s statefulset files are for a single-node cluster. If you deploy this in a multiple-node cluster the master, data, and client pods may end up in different nodes and they will not be able to communicate effectively. Errors such as 'master_not_discovered_exception' may show up.

For multiple-node cluster, use the files suffixed with "-node-affinity". Node affinity will force the deployments and statefulset to be deployed in a particular node (as seen in these k8s manifests) and Priority Class will prevent eviction of pods. Look for a label that is unique to a particular reliable node you want Elasticsearch installed in, and replace them with what is in the "-node-affinity" suffixed k8s files.

### Generate a password and store in a k8s secret:

We have enabled the xpack security module to secure the cluster, now execute the command to initialize the passwords: bin/elasticsearch-setup-passwords within the data node container to generate default users and passwords.  

To manually create passwords for all the Elasticsearch accounts (including account 'elastic'), run this command in the shell:  
kubectl exec -it $(kubectl get pods -n logging | grep elasticsearch-data | sed -n 1p | awk '{print $1}') -n logging -- bin/elasticsearch-setup-passwords interactive

To autogenerate passwords for all the Elasticsearch accounts (including account 'elastic'), run this command in the shell:  
kubectl exec -it $(kubectl get pods -n logging | grep elasticsearch-data | sed -n 1p | awk '{print $1}') -n logging -- bin/elasticsearch-setup-passwords auto -b  

Note the elastic user password and add it into a k8s secret like this:  

kubectl create secret generic elasticsearch-pw-elastic -n logging --from-literal password=<secret_value_not_base64_value>

This secret containing the password will be used to help Kibana, Fluentd, APM service and any other service that will need to connect to Elasticsearch to authenticate to it. In the Kibana, Fluentd, and APM in this repository, the Elasticsearch username is hard-coded to the appropriate env variable. 

## Install Kibana:

cd into the directory kibana and run the following command:  

kubectl apply  -f kibana-configmap.yaml \  
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

Login to the Kibana UI with the elastic account credentials. Navigate to Analytics > Discover

Create a data view with Index pattern “logstash-*”, select “time” in the Timestamp field name option, then save the data view.

![image](https://github.com/osygroup/kubernetes_logging/assets/46828049/940fb9ca-7ace-4f10-8482-a7eb1de604d3)

You can filter logs of pods using namespace, label metadata, container metadata, pod metadata, host(node) etc.
Examples of Kibana query (KQL) using pod label:

kubernetes.labels.app.keyword: "aaasapigatewayinternal" 
log: "<keyword>" and kubernetes.labels.app.keyword : "aaaspaymentgateway"

You can query for logs, save a query search result and share the result (which saves the search result as a csv file).

In Kibana version 8, there is a viewer role that you can assign for read-only accounts.


## Setup Index Lifecycle Management (ILM) to delete old Fluentd logs

Index Lifecycle Management is set up to help automatically delete old Fluentd/Logstash logs (indices) based on age.
From the Kibana UI, navigate to Management > Dev Tools.
In the console, copy and run the following scripts:

#### Create a policy (where min_age is the highest age for a log/index):
```
PUT _ilm/policy/deleteOldIndices
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {}
      },
      "delete": {
        "min_age": "5d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

#### Assign new policy to existing logstash indices:
```
PUT /logstash-*/_settings?pretty
{
  "lifecycle.name": "deleteOldIndices"
}
```

#### Create template for logstash indices:
```
PUT /_template/logging_policy_template?pretty
{
"index_patterns": ["logstash-*"], "settings": { "index.lifecycle.name": "deleteOldIndices" }
}
```

## Install APM:

cd into the directory APM and run the following command:  

kubectl apply  -f apm-configmap.yaml \  
-f apm-service.yaml \  
-f apm-deployment.yaml

Use an Ingress to make the APM server accessible publicly if the service to be monitored is not on the same cluster.

This [documentation](https://medium.com/@bibinkuruvilla/elk-elasticsearch-logstash-kibana-stack-and-elastic-apm-in-kubernetes-7183d871de4c) shows how to set up integration for APM in Kibana under the section 'Next, set up APM'.
In the Server configuration section, you can use apm.logging.svc.cluster.local:8200 for the Host and URL if the service to be monitored is on the same cluster, else use the URL from an ingress.

To install the APM agent on a service to monitor and send performance metrics to the APM server, follow the instructions on Observability > APM page on Kibana.

More useful documents on APM:

https://www.elastic.co/guide/en/observability/current/traces-get-started.html#traces-get-started
https://www.elastic.co/guide/en/observability/current/apm-secret-token.html
https://www.elastic.co/guide/en/observability/current/apm-agent-tls.html
https://www.elastic.co/guide/en/apm/attacher/current/apm-attacher.html
https://www.elastic.co/guide/en/observability/current/apm-directory-layout.html#_apm_server_binary_2
