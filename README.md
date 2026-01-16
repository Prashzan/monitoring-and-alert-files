## How to deploy the different parts of prometheus in kubernetes cluster?

### Using an operator (manager of all prometheus individual components)-----------

### Deployment and stateful set manages the pod replicas

### Operator manages the combination of all components as one unit


### --------------- Using Charts to deploy the operator ----------------- (most efficient)

#### Helm -- will do initial setup
#### Operator -- Manages the running setup

### Demo

-- First we deploy microservices application on kubernetes cluster (eks)

-- Then we deploy prometheus stack which will monitor our kubernetes cluster and our microservices application inside the cluster.

To create eks cluster with default setting
```
eksctl create cluster
```
This will give us cluster with 2 worker nodes

Deploy microservices inside cluster
```
kubectl apply -f config-microservices.yaml
```

### Deloy prometheus monitoring stack using Helm charts

### Deploy Prometheus Operator Stack
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    
### We want to install prometheus into dedicated namespa ce 
    kubectl create namespace monitoring
    
### Installing prometheus chart, we are calling chart name monitoring in monitoring namespace
    helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
    
### To check prometheus pods, wheather they are running
    kubectl --namespace monitoring get pods -l "release=monitoring"
    or
    kubectl get all -n monitoring    
    
    helm ls

[Link to the chart: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack]

### We will see 3 deployments
    1.grafana
    2.Prometheus operator
    3.kube-state-metrics (dependency of this helm chart, scrapes of deployment, sts, pods, svc inside cluster and makes it available for prometheus to scrape, we are getting kubernetes infrastructure monitoring out of the box)
    
### We will see daemonset of node exporter
    Demon set is a component which will run on every single workernode of kubernetes
    Node exporter what it does is, it connects to server itself, and translates the wokernode metrics (cpu usuage) into prometheus metrics that can be scraped.
    
### Big Picture -- We have set up the monitoring stack that monitors different parts, and in addition to that we are getting out of the box monitoring configuration for your k8 cluster,so, worker nodes and stats on worker nodes are being monitored and also k8 components like pods, deployment, replica, stateful set are also being monitored. So, where does all this out ofthe box configuration comes from ?

### we have configMaps, secret basically every configuration for grafana, prometheus etc
    kubectl get configmap -n monitoring
    kubectl get secret -n monitoring  (these will include certificates, username, passwords for diff UI)

#### All these are managed by operator, and these includes all the information how prometheus stack will connect to default metrics, scrape the information and do stuff 

### crd are also created from the stack (custom resource definitions) extension of kubernetes API
    kubectl get crd -n monitoring
     
### We can also check components inside prometheus, alertmanager, operator 
    kubectl get statefulset -n monitoring (this commands list alert manager and prometheus which are used below)
    
    kubectl describe statefulset prometheus-monitoring-kube-prometheus-prometheus -n monitoring > prom.yaml
    kubectl describe statefulset alertmanager-monitoring-kube-prometheus-alertmanager -n monitoring > alert.yaml
    
### To get the Operator
#### list all deployment including grafana, operator and kube-state-metrics
    
    kubectl get deployment -n monitoring 
    
    kubectl describe deployment monitoring-kube-prometheus-operator -n monitoring > oper.yaml
    
#### Inside the prom.yaml, Mounts= where prometheus gets its configuration data
#### configuration file = what endpoints it should scrape ?
#### Prometheus has all the Address of appliation that expose /metrics endpoint, it knows to where to get them from
#### It also also rules file, for example alerting rules that states when cpu usuage spikes to certain %, then send out to email.. These rules are also configuration files

### There are also side container that helps in reloading when configuration changes (config-reloader, this has access to prometheus end point and config-reloader manages rule file)

### Config files comes from secret and rules files comes from configmap

### Important =  Where does the configuration file and rules file come from ?
    They are also part of stack, out of box we get default prometheus configuration files and default rules file. (config files comes from secret and rules files comes from configmap)
    kubectl get secret (we will see list of secrets of which one of which is used in below command i.e. prometheus-prometheus-kube-prometheus-prometheus)
    
    kubectl get secret prometheus-prometheus-kube-prometheus-prometheus -o yaml > secret.yaml
    
    kubectl get configmap prometheus-prometheus-kube-prometheus-prometheus-rulefiles-0 -o yaml > confg.yaml
    
    
### Operator is the name of the container itself (i.e. kube-prometheus-stack)
### We show know how to add/adjust alert rules and how to adjust prometheus configuration so that we can add new end point for scraping


### ---------------------------- Decide what to monitor ----------------------------

#### We want to observe any anomalies like high load, cpu spikes, insufficient storage, unauthorized requests 

### --------------------How do we get this information above ?--------------------
    We get it from prometheus WebUI

    kubectl get svc -n monitoring (this gives us service/monitoring-kube-prometheus-prometheus which is an internal service which gives us prometheus UI)

### ----------------------- To access the internal service from local machine, we have to do port forwarding --------------------

    kubectl port-forward service/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090 & (& for running in background)
    
    so 127.0.0.1:9090 copy this and paste in browser, we will see prometheus UI
    
## Prometheus UI
-- What targets is Prometheus monitoring (inside status navigation, there is target)
this mean if we want to monitor the application like redis, and its not in the target list, we would have to add target to prometheus monitoring target list so that we start get data from them.

-- the stack that we deployed using helm, already includes application that let prometheus scrape data like node-exporter and kube-state-metrics

### ------------------- we want to know Is the data available for the target we want to observe  -------------

### configuration is inside status navigation, There are jobs in configuration file
    Instance = An endpoint you can scrape
    Job = Collection of Instances with the same purpose
    
    there is label name job, which groups them together
    
    
### --------------------------------- Grafana ---------------------------------------

    kubectl port-forward service/monitoring-grafana -n monitoring 8080:80 & (to access in browser)
    
    credential for grafana
    username - admin
    password - prom-operator
    
### Dashboard is a set of one or more panels, we can create dashboards, organized into one or more rows, row is a logical divider within a dashbaord, rows are used to group panels together.

### To get CPU spikes and access in grafana dashboard. for this we will deploy simple pod in our cluster, an image that executes curl commands, this will simulates an application inside the cluster making bunch of request to another application inside the cluster, so we will see the total load

    kubectl run curl-test --image=radial/busyboxplus:curl -i --tty --rm
    
    we are inside terminal now.
    
##### create a script which curls the application endpoint. The endpoint is the external loadbalancer service endpoint
    for i in $(seq 1 10000)
    do
      curl ae4aee0715edc46b988c6ce67121bf57-1459479566.eu-west-3.elb.amazonaws.com > test.txt
    done
    
####  to execute
    chmod =x test.sh 
    ./test.sh
    
### Data Sources (prometheus and alertmanager are the only data sources by default, we can add new data source so that it can visualize the data). Grafana supports many data sources.

    
### (IMP) - Prometheus Operator lets us create custom kubernetes components defined by crd's to create alert rules so operator go to prometheus and say hey by the way there is a new alert rule that you need to pick up or load in and added to your configured list of alert rules. This means if we didn't have prometheus stack running on kubernetes cluster using prometheus operator we would have to go to prometheus config file and we had to add rules to that file and reload the prometheus application to get that new rule. that how this works outside kubernetes cluster. Prometheus operator extends Kubernetes API and lets us create custome kubernetes resources, in background operator does this thing


### -------------------- To execute the alert files and apply alert rules -------------------

    kubectl apply -f alert.rules.yaml
    
    kubectl get PrometheusRule -n monitoring (you will see main-rules file there)
    
### We will see 2 container with prometheus i.e. prometheus itself and side car container (config reloader)   
    kubectl get pod -n monitoring 
    
### To make sure prometheus reload the configuration with new alert rules. One is prometheus itself and other is config reloader

    kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring

    kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c config-reloader  
    
    kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c prometheus
    

#### Create cpu stress
    kubectl delete pod cpu-test;
    
    This is converting docker run command into kubectl run command. the docker command was 
    docker run -it --name cpustress --rm containerstack/cpustress --cpu 4 --timeout 60s --metrics-brief
    
    kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 60s --metrics-brief
    
    kubectl get pod (we must see cpu-test running)
    

### (Important) Prometheus fired the alert to alert manager (firing status), it handed over alert to alert manager. Now alert manager is the one that needs to send out an alert to email or slack etc.


### Alert manager is also part of a prometheus stack and will be the one that will send out alert describing what happened to cluster through mail or slack or different channels.

### Access Alert manager UI
    kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093:9093 &
    
### Display alert manager default rules files
    kubectl get secret alert-manager-monitorin-kube-prometheus-alertmanager-generated -n monitoring -o yaml | less
    
    
### To apply configuration rules for sending email when 2 of the alert fires from alert manager
```
kubectl apply -f email-secret.yaml
```

```
kubectl apply -f alert-manager-configuration.yaml
```

```
kubectl get alertmanagerconfig -n monitoring
```

### To check the logs of alertmanager config reloader
    kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c config-reloader 
    
### Again make cpu stress high to check if the mail was sent or not
    kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 60s --metrics-brief   
    
### To debug the email receiver, to troubleshoot authentication problems
    kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c config-reloader 
    
### --------------- How do we monitor 3rd party applications like redis with Prometheus ---------------

### We actually have Exporters for the 3rd party services.. An exporter is a application that connects to service like redis for example and gets metrics data from that redis service, then exporters then translates these metrics into time series data format that prometheus can understand and after that exporter will expose these collected and translated metrics on its own /metrics endpoint where prometheus can scrape them
  
### When we deploy exporter in the cluster, we need to tell prometheus that there is an end point you need to scrape, and for that there is custom kubernetes resource from the monitoring API called service monitor. Service Monitor Resource needs to be created together with the exporter, that will tell prometheus, there is a new endpoint with metrics data at this specified address
    
### How are we gonna deploy redis exporter in our cluster?
####  For these we gonna use Helm chart that deploys redis exporter application plus all configuration that we needs for this application so that it can run inside kubernetes cluster

    
### To get all the service monitors in cluster
    kubectl get servicemonitor -n monitoring

### We gonna see that each service monitor has these label called release:monitoring so that they can register with prometheus
    kubectl get servicemonitor monitoring-kube-prometheus-alertmanager -n monitoring -o yaml | less
    
### To get service name for redis
    kubectl get svc | grep redis
    Should display (redis-cart) which is used in redis-values.yaml. We basically created redis-values.yaml file here
    
### Redis is not password protected that we are running. anyone can connect to it within the cluster. if it was password protected, we need to provide username and password for the redis exporter as well. for that we need to configure env varibales. (see prometheus redis exporter)

### Deploy Redis Exporter
    helm repo add prometheus-community https://prometheus-community.github.io/helm-chartso 
    helm repo add stable https://charts.helm.sh/stable
    helm repo update

    helm install redis-exporter prometheus-community/prometheus-redis-exporter -f redis-values.yaml (deployed in default namespace)
    
### To see redis service monitor in default name space
    kubectl get servicemonitor
    
### Now we create redis-rule.yaml (redis alert rules)
    kubectl apply -f redis-rule.yaml
    
    kubectl get prometheusrule

    



 
        


    
    





