<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
    </li>
    <li>
      <a href="#deploy-elastic">Deploy elastic</a>
    </li>
    <li>
      <a href="#deploy-filebeat">Deploy filebeat</a>
    </li>
    <li>
      <a href="#deploy-kibana">Deploy kibana</a>
    </li>
  </ol>
</details>

<!-- ABOUT THE PROJECT -->
## About the project
Here I want to deploy a minimalistic stable version of elk running on 
minikube. These are the components I want to deploy:
###  Logging Stack
- Kibana is then used to visualize the data stored in Elasticsearch. It provides a user-friendly interface to interact with the Elasticsearch data and includes features for data exploration, visualization, and dashboarding.
- Elasticsearch would then index this data. Once indexed in Elasticsearch, the data is searchable and available for analysis
- Filebeat would be installed on the servers you wish to monitor. It would tail the log files, line by line (as they grow), and ship the data to Elasticsearch.

<!-- GETTING STARTED -->
## Prerequisites
1. Minikube
To run the elk stack stable on the single node minikube cluster, I decided to go with 4G ram for minikube.
This is how to configure it, alternatively you add the memory and cpu args to the minikube start command 
```bash
minikube config set memory 4096
minikube config set cpus 2
minikube start
minikube config view
```
2. Add helm repository
```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```
3. Download charts (you can use --version to specify a specific version, I am using 8.3.1)
This is nice to see the default values and example configurations which can be found in the examples folder 
```bash
helm pull elastic/elasticsearch --untar
helm pull elastic/kibana --untar
helm pull elastic/filebeat --untar
```

<!-- DEPLOY ELASTIC -->
## Deploy elastic search:
```bash
# deploy elasticsearch
helm install elasticsearch elastic/elasticsearch -f ./values-elastic.yaml --namespace logging --create-namespace
# upgarde elasticsearch when values.yaml configurations are changed
helm upgrade elasticsearch elastic/elasticsearch -f ./values-elastic.yaml --namespace logging
```
4. Create a temporal pod in namespace and port forward to access elasticsearch service
   and check if is running using the credentials from secrets\elasticsearch-master-credentials
```bash
#if this works we know that the elasticsearch service an be reached in the namespaces by the url
kubectl -n logging run my-shell --rm -i --tty --image alpine/curl -- /bin/sh
curl -k -u elastic:${ELASTICSEARCH_PWD} https://elasticsearch-master:9200/_cluster/health?pretty=true
#This can make elasticsearch be accessed from the local terminal
kubectl -n logging port-forward svc/elasticsearch-master 9200 
curl -k -u elastic:${ELASTICSEARCH_PWD} https://localhost:9200
```
4.1 Interessting Elasticsearch URLs for debugging
```bash
#list idexes
kubectl -n logging exec elasticsearch-0 -- curl -XGET 'http://localhost:9200/_cat/indices?v'
#show current data of index in elasticsearch
kubectl -n logging exec elasticsearch-0 -- curl -XGET 'http://localhost:9200/{$INDEX}/_search?pretty'
```

5. Deploy Kibana and config user/pwd for elastic in helm charts
```bash
helm install kibana elastic/kibana -f ./values-kibana.yaml --namespace logging
```
6. Check Kibana access to elasticsearch
```bash
kubectl -n logging port-forward pod/kibana-kibana-8647c65456-hb8k4 5601:5601
curl -k -u elastic:${ELASTICSEARCH_PWD} https://elasticsearch-master:9200/_cluster/health?pretty=true
```

7. Deploy filebeat
```bash
helm install filebeat elastic/filebeat -f ./values-filebeat.yaml --namespace logging
# watch all containers come up
kubectl get pods --namespace=logging -l app=filebeat-filebeat -w
```