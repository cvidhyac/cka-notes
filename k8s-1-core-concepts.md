# k8s core concepts

- discuss high level overview of the k8s components
- refresher for basic k8s objects like replication controllers, deployments and services.

## k8s Components

* etcd - key value data store that stores the status of every action in the k8s cluster
* services - load balancing
* k8s controllers (Replication controller, Admission controller) - various k8s functionalities
* kube-scheduler - schedules workloads on to worker nodes based on taints/tolerations, resource constraints etc.,
* kube-proxy - network communication between different k8s components like etcd, services
* kubelet - present in every worker node - communicate with control plane on status, capacity etc.,
* k8s API Server - orchestration between different components

## etcd features

* document store - key value, does not support complex queries - distributed and reliable
* download binary install it - it comes with a `./etcd` executable.
* different versions of the command is available
* Uses RAFT consensus protocol
* `etcdctl` API changes - (v2 does not support txt, v3 does)

Every change to cluster is noted in etcd server. Only after it is updated in etcd server, a change is considered
complete.

Two ways -

* from scratch
* via kubeadm tool

Set up etcd.service on master node, also need to set up `--cert`, `--cacert` and `--key` for certificates

## kube-api server

- orchestrator for all k8s components

Workflow when a kubectl command is run to create a pod:

- kube-api server receives the command
- authenticates it
- run a curl POST request that creates the POD object
- once pod object is created, the details are saved in etcd. At this point the object is saved without a node
  assignment.
- kube-scheduler watches for all the activity in the cluster. It notices a new pod object has been created, it proceeds
  to identify a node where the pod can be scheduled. And returns the node details to the kube-api server
- kube-api server reaches out to the kubelet on the worker node, then kubelet helps provision the pod on the node.
- once the pod is created successfully, kubelet informs kube-api server back, and the node details are saved back
  to etcd.

Whenever there is a change in lifecycle, the same cycle repeats.

There are two ways to provision kube-api server. It is set up with defaults if kubernetes cluster is bootstrapped by
the default kubeadm utility. However, if the cluster was spun up the hard way, it needs to be exposed as a systemd
service. There are a number of parameters that need to be set up. Refer k8s docs for details on this one.

## kube-controller manager

Brain behind a lot of activity in the cluster.

Two main responsibilities:

- Watch status
- Remediate the status to bring the current state to desired state.

- There are many controllers in k8s, each controller has its own responsibility.
- Node controller monitors status of nodes. How does it do this?
    * Node controller monitors status of nodes, it communicates with the kubelet in every node to get health.
    * If health is not returned every 5s, then the node is marked unreachable for the next 5 minutes. Within this time
      period, if the node does not return status back, then all the workloads in this unhealthy node is re-provisioned
      on other healthy nodes.

All controllers are packaged together in kubernetes controller manager package. Default settings can be overridden
when the installation is run, there is also `--controller` option to select which controllers to run.

Two ways to install it -

- kubeadm sets this up with defaults,
- if cluster was bootstrapped the hard way, then it needs to be set up as a systemd service after installing the
  binary.

## kube-scheduler

- Decision maker - does not execute any actions. It analyzes the resources available in the cluster to decide where
  a workload should be scheduled.
- How does it do this - 2-step strategy - first it filters out the nodes where it cannot be scheduled due to
  taints/toleration, node affinity rules, resource constraints etc.,. Of the ones that pass the filter check, a ranking
  strategy is applied. The node with the highest rank is the one that kube scheduler returns to kube-api-server

Two ways to run kube-scheduler:

- If kubeadm utility was used to bootstrap the server, kube-scheduler with defaults already runs as a service in
  cluster and can be viewed in the list of pods in `kube-system` namespace
- If cluster was set up the hardway, then binary must be installed and set up as a systemd service on the master node.

## Services

- Virtual network of pods that contains the rules of how traffic should be routed
- When IP is not static, services will query the IP Table rules in each node to find out the latest IP to which
  traffic should be routed.

## kube-proxy

- Facilitates the network connectivity between all k8s resources in a cluster
- Whenever new services are added, kube-proxy creates and updates the IP Table rules.
- There is an IP Table in every node

## kubelet

- Owns the execution actions on the worker node (like a captain on a ship)
- Executes the actions sent by kube-api server and returns status back. Also is in close communication with the
  kube-api server on node status and health.

## Replication controllers / ReplicaSet

Replication Controller is the older term, ReplicaSet is the newer way.

- Runs multiple pods in one container for two purposes - resilience in case one goes down, load balance the traffic
  across all pods of the same type in the deployment.
- difference between replication controller and replicaset - `ReplicaSet` requires a `selector` in the spec definition.
- Note: `selector` is not a required field. when `selector` is not provided, then k8s assumes the selector to be same
  as `metadata.labels` in the `ReplicaSet` spec definition. When specified, the specified labels are used.

```shell
k create replicaset -f my-replicaset.yaml
k scale --replicas=6 replicaset myreplica
k scale --replicas=6 -f my-replicaset.yaml ---> Note that this command will increase replica but not update yaml file.
```

A replicaset spec definition contains a pod template + replicas parameter + selector

## Deployment

A deployment is a higher level abstraction on top of replicaset. From a yaml perspective, it closely resembles the
`ReplicaSet` kind and only difference is the labels parameter and uses a kind called `Deployment`.

When applied, th a `Deployment` creates its own object, a `ReplicaSet` and the number of replica pods requested in
the yaml.

## Services

* Helps expose pods outside node
* 3 types of services: NodePort, ClusterIP, Load balancer

### Services: Type NodePort

* 3 attributes:
    * Target Port : resides on the pod
    * Port: Resides on the service object
    * Node Port: Resides on the node itself
* Of the 3 port attributes only the `port` is required attribute. When `targetPort` is not specified, then it is
  assumed that `port` == `targetPort`. A random `nodePort` within `32767` is assigned as nodePort.

### Services: Type ClusterIP

- IPs will not be static between application tiers (db, frontend, backend). Pods are continuously created / terminated.
  We need a single interface to group pods in a group. This interface is called clusterIP, it allows the set of pods
  to be accessed via either its `clusterIP` or its service name.
- The spec is very similar to `nodePort` type, except that node port attribute is not present. Selector from pod
  template allows the grouping. This is the default type in k8s.

### Services: Type LoadBalancer

* k8s has out-of-box integration with AWS, Azure and GCP to use the hyperscaler provided load balancers.
* If app runs on the major clouds, by specifying type as `LoadBalancer` then it would help balance load automatically
  without any additional config.

## Namespaces

- default namespace
- Resources are limited per namespace
- While connecting to a service in another namespace, the service must be referred with the service name along with
  the name of the namespace. This is because when a service is created, a dns entry with the namepace name is added to
  etcd.

For example, if `my-db-service` is located in `dev` namespace, it should be referred to as by its DNS entry -
`my-db-service.dev.svc.cluster.local`.

## Imperative v Declarative style in k8s

- Short commands are imperative - they specify exactly what needs to be done - eg. scale the pod to 5 replica
- Complex commands benefit from declarative style - eg: create a multi-pod deployment

K8s supports declarative programming by the use of YAML object configuration files.

Declarative style helps version the command and send it for code review and save the state for tracking.

A Change made by `kubectl edit` is not saved back to file, hence we should avoid imperative commands as much as
possible.

instead of using `kubectl edit`, go for MR review on the yaml file then use `kubectl apply -f` with force if needed
in case object needs to be replaced.

### kubectl explain command

`kubectl explain pods.spec --recursive`

explains the fields when accessing k8s documentation is not possible.

## How does kubectl apply command work?

- It applies the object configuration file (yaml) file to the k8s as a "Live Object"
- Then, the current state of the object in yaml is converted and saved as a JSON resource on the live object itself
  as `kubernetes.io/applied-configuration`.
- When the next kubectl apply command is requested, it uses the last applied configuration as a reference to decide
  if the given request should be executed or not. Some updates may require deletion and recreation of resources or force
  while others are allowed as an update.
- Saving the last applied configuration is a special feature in this command not available in other imperative commands.
