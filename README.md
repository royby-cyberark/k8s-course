My notes for this course:
https://www.udemy.com/course/kubernetes-microservices

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

# Cluster Monitoring
* Allocate more resources, working with Docker desktop on Darwin: in the docker desktop ui, set resources (cpus, RAM), then run minikube with: `minikube start --cpus <# of cpus> --memory <Size in MB>`
* Show pods resources consumption and other info: `kubectl describe nodes`
* 
