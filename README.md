My notes for this course:
https://www.udemy.com/course/kubernetes-microservices

See branches for:
* local-cluster
* eks-cluster
* prometheus-stack

IMPORTANT! Before deploying anything, replace ip with your own for any IP under `loadBalancerSourceRanges`. this will allow you to access those services that are only exposed to that address.
# Course project startup
* `minikube start`
* `minikube service --url fleetman-queue`
* `minikube service --url fleetman-webapp`

# Performance
You might need to up the minikube resources.
On macos, I set it in docker desktop - preferences -> Resources -> Memory:4GB, CPUs:8
But also, start minikube with:
`minikube start --cpus 4 --memory 4096` (or more)

# Running Minikube on Darwin

When running minikube in Darwin (macos) with the docker desktop driver, you won't be able to access services using NodePort (or any?) using the minikube ip directly. 

This will help you:

`minikube service --url <service name>`
(I ran `minikube service --url fleetman-webapp`)

_"Because you are using a Docker driver on darwin, the terminal needs to be open to run it"_

See issue: https://github.com/kubernetes/minikube/issues/11193

# Debugging services
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/

----

Queue image:

`richardchesterwood/k8s-fleetman-queue:release1`

# Commands
## minikube
* `start`, `stop`, `status`, `ip`, 
* `service --url <service-name>` - required in darwin to be able to access the service ip

## kubectl
* get
  * `kubectl get all`
  * get pods by label app=hostnames: `kubectl get pods -l app=hostnames`
  * get service endpoints: `kubectl get endpoints <service>`
  * get pods (short) with labels: `kubectl get po --show-labels`
  * get pod show some label: `kubectl get po --show-labels -l release=0-5`
* `kubectl apply -f <filename>.yaml` - apply settings
* `kubectl apply -f .` - apply all yamls
* `kubectl describe [pod|service|svc] <name>` - describe something
* `kubectl exec <pod name> -- <command>`, example: `kubectl exec webapp -- sh`
* `kubectl exec -it webapp -- sh` - run shell with terminal
* Delete replica set: `kubectl delete rs webapp`
* get rollout status: `kubectl rollout status deploy(ment) <deployment name, eg. webapp>`
* rollout history: `kubectl rollout history deployment webapp`
* delete all resources created from a yaml file/all files: `kubectl delete -f <file.yaml>`, `kubectl delete -f .`
* get pod logs (only pods have logs): `kubectl logs [-f] <pod-name>` (-f to follow)
* get all pods from all namespaces: `kubectl get all --all-namespaces`

# Replica sets
* you can specify how many replicas you desire and k8s will make sure those are always up
* When we upgrade we might have a short down time

# Deployments
* Similar to replica sets with automatic rolling updates
* Creates and manages replica sets
* We can create a zero down-time deployment with gradual deployment
* see rollout status with above commands. can also rollback. 

# Rolling back rollouts
* Get rollout history: `kubectl rollout history deploy(ment) webapp`
* `kubectl rollout undo deploy(ment) webapp [--to-revision=2]`
Optionally specify the revision, but by default it rolls back one revision

# K8s namespaces
Logically separate your resources with namespaces. 
E.g. to get K8s` DNS service:
* `kubectl get all -n kube-system`

# Networking/DNS
Built in DNS (find with get all with the kube-system ns)
Find your services by, for example, `nslookup <servicename>`

# Course Microservices Resources

* Github repo: https://github.com/DickChesterwood/k8s-fleetman

# Data persistency 

## Volumes

We can add volumes to containers, see mongo-stack.yaml
We can use local disk or many other options like cloud storage (EBS, Azure objects) and much more.
The volume configuration is seperate from the mount settings.

https://kubernetes.io/docs/concepts/storage/volumes/

We added the following to our deployment spec, which is nice for dev, but for prod we will do something like store on EBS or some other cloud storage.

`hostPath` - this means that its a local path (in docker desktop, on the VM running minikube)

`type: DirectoryOrCreate` - will create the dir if not existing

```
 volumes:
   - name: mongo-persistent-storage
     hostPath:
       path: /mnt/some/directory/structure
       type: DirectoryOrCreate
```

## Persistent volume claims
You probably don't want to hard code this into the deployment yaml, so that you can more easily replace the volume specs.
So its better to use claims which are pointers to another config (best in its own yaml)

* View peristent volumes: `kubectl get pv`
* View peristent volume claims: `kubectl get pvc`

# Deploy cluster with minikube
* `minikube start --cpus 8 --memory 5500`
* you get: "Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default"
* `kubectl apply -f .`

# EKS - K8s on AWS
## Creating the EKS cluster
* Check kubectl supported version in the EKS dashboard (create new but don't follow through, just get the default version of kubectl)
* Install the right version of kubectl
* Install eksctl
* Create cluster: `eksctl create cluster --name <cluster-name> --nodes-min=3`
* WARNING! If you apply your yamls, as-is the LB service will create an opened SG. to avoid this, make sure you include spec.loadBalancerSourceRanges to allow only specific ip(s) - https://kubernetes.io/docs/concepts/services-networking/_print/

* Then apply your config: `kubectl apply -f .`

## Destroying the EKS cluster
* TRY: to un apply by: `kubectl delete -f .`
* Run: `eksctl delete cluster <cluster-name>`
* Verify destoryed: 
  * ec2 instances (is auto ternimated)
  * ec2 dashboard, Load balancer - verfy deleted (is auto deleted)
  * In EKS, clusters - verify cluster is deleted (is auto deleted)
  * ec2 dashboard, EBS, volumes
    * **NOT AUTO DESTROYED! delete the kubernetes-dynamic-pvc-XXXX (how do you know which one?) - it will be in state=available**
      * get leftover volume: `aws ec2 describe-volumes --filters Name=tag-key,Values=kubernetes.io/cluster/<cluster-name>`
      * delete this volume: `aws ec2 delete-volume --volume-id <vol-id>`
    * **NOT AUTO DESTROYED! if you're also ran an ELK stack, you will see an additional 3 volumes**
    * You can search by tag values, use the cluster name

## Changes on EKS
* Updated the storage.yaml to specify the aws-ebs storage
* In services.yaml, for the fleetman-webapp service, we now use LoadBalancer as the type - we can do this only in a cloud environment where we have the load balancer. 
  * Removed the NodeIp and updated the type to be LoadBanacer
* for the queue service (and API gw if relevant): we removed the NodePort, and the node type to a ClusterIP so that the queue is only accessible from the cluster.

## EKS - Node failure survival
* To know what nodes your pods are running on, run: `kubectl get pods -o wide` - and see what instances each pod's running on.
* In case a node goes down (e.g. terminated), then a new node instance will be brought up. this is done by the auto-scaling group.
* If we have an app running only on one pod, then we are at the graces of the auto-scaling group.
* **k8s** will NOT auto-balance the pods between nodes, if one node goes down k8s will start the pod on existing ones, even that auto-scaling might bring up another node
* in our workloads.yaml we use deployments - where we specify the replicas number - how many instances of the pod are running at a time
* We update our webapp replicas to 2, 3 or whatever your requirements are.
* What about other pods? the queue is stateful, we can't simply replicate it. **STRIVE FOR STATELESS PODS!** - what to do?
  * look for replication options for the queue (ActiveMQ does that), same for the db
  * better - look for your cloud provider (AmazonMQ/SQS), same for db

# Cluster Monitoring
* Allocate more resources, working with Docker desktop on Darwin: in the docker desktop ui, set resources (cpus, RAM), then run minikube with: `minikube start --cpus <# of cpus> --memory <Size in MB>`
* Show pods resources consumption and other info: `kubectl describe nodes`
* For installing the ELK stack into our cluster, we took the course files: elastic-stack.yaml, fluentd-config.yaml. it was previously in https://github.com/kubernetes/kubernetes/tree/master/cluster/addons, not sure where they went.
* IMPORTANT! I added the loadBalancerSourceRanges like in services.yaml to avoid exposing the LB address
* apply the two files
* kubectl get pods - you only see your pods (remember namespaces), elk is deployed to the kube-system ns, use this: `kubectl get pods -n kube-system`
* wait until all pods in kube-system are running.
## Monitoring using an Elastic stack - Elasticsearch, Fluentd (or Logstash), Kibana
### Getting started with Kibana
* get kube-system services: `kubectl get svc -n kube-system`, kibana has an external IP
* see kibana LB ingress address: `kubectl describe svc kibana-logging -n kube-system`, note the pod. you can also see this in the AWS LB page, and find the port under the "Listening" tab.
* Open kibana on the lb address with port 5601

### Setting up indices
* Click on "setup index patterns"
* We need to give Kibana the name of the index. our yaml which uses fluentd, will have by default a daily rotating index with a prefix of "logstash", you can also see this under "discover" in Kibana.
* select @timestamp for the time filter
* Create index
* Open discover to see the logs and the search engine
* This includes ALL logs, including the system logs.
* To filter and see only our logs, its best to use namesapces: add a filter by looking up the kubernetes.namespace_name, expand it and from the "default" namespace (our namespace), click the + to add a filter. if you can't see it, you probably don't have errors in this timeframe, try to expand it to a longer one.
* Under visualization you can create graphs, charts, etc. 

## Monitoring with Prometheus and Grafana

### Settings up the EKS monitoring stack (prometheus, grafana)
We want to install the Prometheus stack on our cluster, it is here: https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack, and generally you should follow the instructions there for any real system.

But, we have a ready to apply yaml for this for testing purposes. this includes the custom-resource-definitions custom_resources_defs.yaml which you wouldn't want to use otherwise. 

* **THIS IS ONLY FOR TESTING PURPOSES! IN REAL LIFE WE WOULD USE A PROPER HELM CHART**
* you MUST apply `custom_resources_defs.yaml` first.
* then, apply `eks-monitoring.yaml`
* everything is deployed is the 'monitoring' namespace. list ns's with: `kubectl get ns`
* list pods/svc: `kubectl get <pod/svc> -n monitoring` - note that no svc has an external ip

* What did we get? (list svc)
  * Prometheus to monitor different aspects
    * svc `monitoring-kube-prometheus-prometheus` is the Prometheus ui (port 9090)
  * Grafana to visualize
  * Alert manager

All the services are deployed as ClusterIP, which means they are not accessible from the outside there are several ways to expose them.

* Here we will first change the ui svc into a LB, but later we will see other ways - updated the monitoring-kube-prometheus-prometheus service to LB as before (with access only to our IP)

* get your LB address:port with `kubectl get svc -n monitoring` and login
* node_load1, node_load5, node_load15 - show the load on all nodes in the last 1, 5, 15 minutes.

### Removing prmetheus LB
* We won't need to actually access prometheus externally, so revert the LB type into a ClusterIP
* apply and verify that its a clusterip now by running `kubectl get svc -n monitoring`


### Getting started with grafana
* find grafana service with `kubectl get svc -n monitoring`
* To allow access externally, we will set it as a LB, but we can also use a NodePort (see next) which is less robust
* update the type of the service "monitoring-grafana" to LB with the usual ip restrictions
* log in and update the default creds
* Clicking on "Home" you get a list of many useful and less useful dashboards, many are focused on k8s. 
  * we will focus on two: "USE Method / Node" - only for one node. "USE Method / Cluster" - for the entire cluster (better for health overview)
  * USE is Utilization-Saturation-Errors and is a standard for monitoring such systems
  * You can also look at k8s specific dashboards, e.g. "Kubernetes/Compute Resources/Pod"
  * Monitor persistent volumes dashboard: "General/Kubernetes/Persistent Volumes"
  * There are also other flavors of dashboards, e.g. "Nodes" similar to USE node

# Using NodePort to access services instead of using LoadBalancer
* Change service type to "NodePort"
* under ports, next to targetPort, add "nodePort" and give it a random high port. make sure it doesn't conflict with previously assigned port.
* Update the security group to allow access to the high port from your ip address.
* How do we know to what ip of which node to access? (we don't know on which node our service is running) - we can use ANY of the nodes public IP in the cluster!
  k8s knows how to redirect you to the right node. 
  **When you make a request to k8s, it doesn't matter to which node you make it, k8s takes care of it**


# Alerting
* Deploy prometheus with a LoadBalancer service so we can access it (or with a NodePort)
* Open prometheus, alerting
## Configure AlertManager with slack notifications
Some docs: https://grafana.com/blog/2020/02/25/step-by-step-guide-to-setting-up-prometheus-alertmanager-with-slack-pagerduty-and-gmail/

Prometheus are using a k8s secret to store the config. there's also config-map which is very similar, but here it uses a secret.

* Our config file is `alertmanager.yaml` file, update it with the slack url and also the channel name.
* List secrets with `kubectl get secrets -n monitoring`, the config one is: `alertmanager-monitoring-kube-prometheus-alertmanager`
* Then get your secret yaml with `kubectl get secret -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager -o yaml`
* The data is your config in base64. To see, you can take it and run `echo <DATA VALUE> | base64 -d`
* We must delete the secret first, before recreating it: `kubectl delete secret -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager`
* Now create the new "secret", by running `kubectl create secret generic --from-file alert-manager/alertmanager.yaml -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager`
  * to create the secret from file you need the `generic` and `--from-file`

### Troubleshooting (also looking in containers)
* Verify webhook is working (curl it)
* Get the secret and verify the values of url and slack channel
* To see if we have issues with our config, we can kubectl logs to the alert manager pod
  * `kubectl get pods -n monitoring`
  * `kubectl logs -f -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0`, if you get an error that you need to specify a container name, then do it (the error will tell you what container names are available - alertmanager, config-reloader) by adding the -c <container name>
    * `kubectl logs -f -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0 -c alertmanager/config-reloader`
    * get container names with `kubectl describe pod <podname> -n <namespace>`

## Alert manager UI
* The service we want is `monitoring-kube-prometheus-alertmanager` (see with `kubectl get svc -n monitoring`)
* Open it with a LB or a NodePort
* A possibly important feature here is silencing (silence button in the ui) with a duration
* We might want to silence in advance on the case of planned maintenance, here we use the "new silence" 
* alerts ARE STILL FIRING! just not alerting anything

## Dealing with the watchdog alerts
* We have a heartbeat alert (watchdog) that we get constantly, but we need an external system to notify us if we DO get an actual error if if the watchdog stopped alerting.
* We need something external, even to our AWS account, region etc. 
* Example vendors: 
  * [Dead man snitch](https://deadmanssnitch.com/plans) - simply get a url and use it for the watchdog, we don't know if its AWS.
  * Being on AWS like us, maybe even in the same region, is not good, since if AWS goes down we get silence. 
  * We could investigate by nslooking the hostname, then the ip and try to identify the source.
* in the alert manager ui, the "silence" is like a snooze button where you silence alerts for a period of time

## Using pagerduty with alertmanager
see `alert-manager/sample_alertmanager_with_pagerduty.yaml`
* it used the built-in configuration for pagerduty, which only takes the API key (see pagerduty_configs in the config yaml)
* It also sends watchdog to dead man's snitch - under "DMS"
* Under route, we specify that matches on alertname: Watchdog, will go to DMS and NOT to the other receivers.
* To avoid bombarding us with the same alert, e.g. is something caused many pods to fail, then we group alerts with the `group-by: alertname`, which means that it will treat all alerts with the same name as the same alert within the group_interval time.
  * group_wait is the wait time in seconds that alert manager will wait after the first alert, and before it starts alerting
  * if for example, we want to wait for autoscale to correct something, we might want to set this value to something higher (e.g. 10 minutes?)

# Requests and limits
* Assuming we know what resources - cpu, memory, our pod needs (which we, as devs, need to determine) - we can specify it in its deployment yaml.
* if a node has not more RAM according to these requests, then it will not schedule a pod to that node, if there are other nodes available with free resource, then it would be deployed there, otherwise it will fail.
* if you are working with minikube for example, you can run `kubectl describe node <master node name>` to see how much memory it has (Capacity), and how much is "allocatable" which is the account available to nodes (Allocatable)
* If we don't have enough resources, our pod will be pending, and describing it will show the problem.
* **THESE ARE JUST HINTS! it will make sure you don't over deploy pods, but it won't limit anything in runtime.**
## Memory Requests
* We added to `workloads.yaml` resources.requests.memory: 300M - this doesn't mean that it uses the full amount, just that it requests this amount to be available to it.
Our process could hog more memory than that. it only means that we make sure the node has enough resources. we can limit it with limits next.

## CPU Requests
* 1 cpu in k8s is 1 vcpu in aws and similar in other cloud vendors
* We added to workloads.yaml resources.requests.cpu: 1 - again, not using the full cpu, just that it verifies that the node has this capacity. our process can still hog more resources. to limit it, we will use limits next
* Usually a pod won't need a full cpu, so we can use fractions like 0.1, 0.5 etc. or milli cpus, e.g 100m (100 milli cpu = 0.1 cpu)

## Limits
* Same syntax as requests, we will add to `workloads.yaml` resources.limits.memory, cpu. they should be equal or higher than the request values.
* Memory limit - if actual memory usage of container exceeds the runtime at limit - container is restarted
* CPU limit - if actual cpu usage exceeds at runtime - cpu will be clamped to the max value for this container (throttling)
* We could for example, set a higher limit to memory to support some rare memory leak which we didn't find yet
* all this uses Linux cgroups (or at least used)


# K8s dashboard
k8s has a dashbaord, its different if you run on minikube, actual k8s or eks. 
you can see and do basically everything here, but usually we do infra as code.

* for minikube its an addon, list addons and its called `dashboard` (in the kube-system ns)
* enable with: `minikube addons enable dashboard`
* show with: `minikube dashb` (its using a proxy so you will see it on the localhost)

# Metric profiling in k8s
* We could monitor a running cluster and get some metric as to how much resources it uses
* We can use the top command (like linux): `kubectl top pod`, `kubectl top node` - but out of the box this doesn't work
* We need to enable the metric server (running in the kube-system ns)
* It's tricker in eks, but locally it's simple - it's an add-on to minikube
* `minikube addons list` - and we will need to enable the `metric-server`
* enable with: `minikube addons enable metrics-server`
* you can see a new pod in the kube-system ns named metric-server
* for at least a minute for it to collect data (see metrics not available etc.) - eventually the top commands will work
  * see how much the containers are actually using.
* You should be able to see the metric-server graph in the dashboard, but in the past there was some issues, probably solved by now.
  * if there's still an issue, you can enable heapster which is the prior solution to metric-server - see the course video for details ("Viewing metrics on the dashboard) - but heapster was deprecated so it's probably not relevant.
* 

## Java apps tuning
* Important to use `-Xmx` to set the maximum heap size. 
* Then we would monitor the actual ram, cpu usage and set reasonable requests values accordingly
it

## Horizontal pod scaling
* We need pods to be stateless so we can scale them horizontal 
* enable metric server: `minikube addons enable metrics-server`
* docs: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
* we can create an autoscaler with cli but then auto generate the yaml for it: (you can autoscale deployment or replicaset)
* example for autoscaling api-gateway:
  * `kubectl autoscale deployment <autoscaler name> --cpu-percent <percent relative to the request you've specified, can be > 100> --min <min replicate number> --max <max replicate number>`

  * `kubectl autoscale deployment api-gateway --cpu-percent 400 --min 1 --min 4`
* get horizontal pod autoscale objects: `kubectl get hpa`
* we need one for each deployments we want to scale
* we can use `kubectl top pod`, `kubectl top node` to check things out
* `kubectl describe hpa api-gateway`
* auto generate yaml: `kubectl get hpa api-gateway -o yaml`
  * create a file with the content
  * clean:
    * in metadata: annotations, creation timestamp, resourceVersion, selfLink, uid
    * in spec: status - is the current status, remove it.
  * apply the new file
* scale down take time so we wont scale up/down rapidly

# Readiness and liveness probes

## Readiness probes
* when we scale up/down it might take just a little time until the container application becomes ready, but the container is ready before that and we might get errors on those scaling up/down events.
* docs (see more params): https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
* We will use a readiness prob - we will tell k8s how test if a pod is ready by giving it some api endpoint to call, only then will it start directing traffic to it. we can also do without an http server - see later
* under spec in the container - see workloads.yaml, api-gateway
* we can see the prob info with pod describe command

## Liveness probes
* configured same as readiness prob, but is called continuously, and if k8s gets errors it restarts the whole CONTAINER
* same params as readiness, like how many failure to get before it restarts the container

## With an http server
* we can do without an http server, with a "liveness command" (or readiness) - https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command
* it runs a command (exec) on the container
* we can aldo do TCP prob - see docs

# QoS and Evictions
## QoS
* pods that has limits - the scheduler will evict such pod if it goes over the limit, and reschedule
* if you set requests and limits you give the scheduler all the info it needs, if you only set requests you give it less, and nothing give it nothing. k8s labels the pods according the their specs (both memory and cpu) - for more details see the docs:
  * Specify equal requests and limits: "QoS: Guaranteed" (both mem and cpu req+lmt are equal)
  * Specify request but NOT limit - memory OR cpu: "QoS: Burstable" 
  * No req, no limit - "QoS: BestEffort"
* You can see the labels by running `kubectl describe pod <pod>`, see the "QoS Class" field
## Evictions
The labels above, help the scheduler decide what to do with each pods when they are under pressure.
The worst thing is if we lose nodes, e.g. due to it running out of memory.
* "QoS: Guaranteed" - the best, if it goes over the limit it will automatically be evicted (uses Linux cgroups) and the node won't suffer. it will then be rescheduled on another pod. **Of course we must take care of the requests and limit values to make sense**
* "QoS: Burstable" - we have only requests here, if we go above the requests. "burstable" means that it allows to go over the requests (in k8s this doesn't mean only for a short time, but just go over the reqs). we could go over the node limits.
  * if the node goes over its resources, scheduler needs to evict something: 1. best effort pods first, 2. then burstable, 3. then guaranteed pods. 
  * pods are rescheduled - restarted and put on a node somewhere.
  * if there are no free pods, and if we're using cluster auto-scaling, then we should have a node created dynamically
  * **this is an automatic priority system**
* **REMEMBER:** This is over simplification of the actual logic of the cluster, see the docs for more info https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/

## Pods and priorities
docs: https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/
* You can add priorities (number 0, 1, etc - higher is higher priority) to pods
* Mainly used for NEW pods that are scheduled - if we already have a scheduled pod and we got a higher priority pod - k8s will evict the previous one (and reschedule it) and schedule the new, higher priority one. 
* Configuration: some yaml with PriorityClass with a numeric value and a label and reference it in your pod def. 
* QoS and priority are related (they should be orthogonal, but they are related), see the docs for more info.

# ConfigMap and secrets
To avoid duplicating things like env vars in our yams, we can define a config map and get it from there, updating it in one place.
* see `database-config.yaml`, apply it
* `kubectl get configmap` (or cm)
* `kubectl describe cm global-database-config`

## Referencing ConfigMap values - I
We have 3 ways to consume those values. 

The worst way (many lines, and we need this for each reference, each var in this case), in your env element:

```
env: 
- name: DATABASE_URL
  valueFrom:
    configMapKeyRef:
      # the name of the config map we defined:
      name: global-database-config
      # the key from our configMap yaml:
      key: database.url
```

  * to check the env var
    * `kubectl exec -it position-simulator-647b8cccf5-zslvs -- sh`
    * `echo $DATABASE_URL`

## ConfigMap values propogation
The changes will NOT be updated in the pods. we recreate the pod (e.g. delete it). 
See docs to see if there is another, newer method for this.

See: https://github.com/kubernetes/kubernetes/issues/22368

A workround can be to treat configMap as immutable, and abandon each configMap on changes, and create a new version (e.g. global-database-config-v1, v2 etc.), then apply the configmap and workload yamls and we get the update.

## Referencing ConfigMap values - II (using envFrom)

Here we get ALL variable from the specified configMap.
all the vars from the config must be valid env vars.


```
env:
- name: SPRING_PROFILES_ACTIVE
  value: production-microservice
envFrom:
- configMapRef:
  name: global-database-config-v2
```

## Mounting ConfigMap as Volumes
Instead of injecting env vars, we can mount the values from the configMap as a file, mounted to a directory using `volumeMount`
mountPath folder is created automatically.

**We set the `position-tracker` (see workloads.yaml) to work with mounts** - see the `volumeMounts` and `volumes` elements

run `kubectl exec -it <position-tracker-xxx> -- sh` and navigate to /etc/any/directory/config

This approach make each env var intoa file. 
A more suitable way might be (in the database-config.yaml):
we need to split the line into several lines with line breaks use the | char, which will have all the following indented part as the content of the file (we also updated : into = for a proper properties file)

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-database-config-v4
  namespace: default
data:
  database.properties: |
    database.url=https://dbserver.somewhere2.com:3306
    database.user=myuser
```

