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


# EKS - K8s on AWS
## Creating the EKS cluster
* Check kubectl supported version in the EKS dashboard (create new but don't follow through, just get the default version of kubectl)
* Install the right version of kubectl
* Install eksctl
* Create cluster: `eksctl create cluster --name <cluster-name> --nodes-min=3`
* WARNING! If you apply your yamls, as-is the LB service will create an opened SG. to avoid this, make sure you include spec.loadBalancerSourceRanges to allow only specific ip(s) - https://kubernetes.io/docs/concepts/services-networking/_print/

* Then apply your config: `kubectl apply -f .`

## Destroying the EKS cluster
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





