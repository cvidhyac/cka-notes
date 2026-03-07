# k8s scheduling

## Different ways of manually scheduling a pod on a node

* k8s scheduler works by looking for the `nodeName` property on the pod objects. Any pod object that doesn't have
  the property would be picked up for scheduling.
* when scheduler is unavailable, the pods will be in `Pending` state until they are picked up.
* When scheduler picks up the pod for scheduling it on the node, it first creates a `Binding` object, which internally
  issues a post request to the binding api and then saved to etcd. Once binding request is successful, the node can now
  create the pod.

### Option 1: Manual scheduling

* The `nodeName` property can be added to the object configuration file BEFORE the pod is created in k8s. This cannot
  be added to pods that already exist on the cluster because they would have been already scheduled.
* The node name if specified in the file is similar to hardcoding the node. The pod will be scheduled to whichever
  node is specified in the file.

### Option 2: Create the Binding Object manually

* Create an object configuration file for the `kind: Binding` specifying the node name as `target`. The binding object
  and pod would share same `metadata.name`
* Once the binding object is applied on the cluster, k8s will automatically update the `nodeName` on the pod live
  object.

## Labels, selectors, annotations gotchas

- Selectors don't have to match the labels of the object, it really should match partially with the pod template labels
- When selector is not specified, it is assumed that it matches all the labels in pod template
- At least one label should match, add all labels for more restricted matches.

Annotations are for record and informational purposes to capture additional information not already captured in labels.

## Taints and Tolerations

* Taints are applied on nodes to prevent workloads from being scheduled on that node for specific reasons
* Tolerations are applied on pods to allow them to be scheduled on nodes with taints.
* `k taint nodes node01 app=blue:NoSchedule`
* There are 3 taint effects possible - `NoSchedule`, `PreferNoSchedule` and `NoExecute`
    * NoSchedule - pod will not be scheduled, guaranteed
    * PreferNoSchedule - pod will not be scheduled, not guaranteed. Existing pods will not be evicted
    * NoExecute - evict the pods already existing without tolerations

## Node Selector vs Node Affinity rules

- Use node selector when condition to schedule a pod to a node does not require complex logic.
- Use Node Affinity Rules when complex logic and operator access is required to schedule a pod to a node.
    - Two types of node affinity rules are available:
        - `requiredDuringSchedulingIgnoredDuringExcecution`
        - `preferredDuringSchedulingIgnoredDuringExecution`
    - In both these types of affinity rules, workloads already scheduled are not impacted. It only applies for new
      workloads pending schedule.
    - Node Affinity rules have more complex operators and label selection options.
- Both Node Selector and Node Affinity rules are applied on pods.

## Resource Limits

- Requests: Minimum guaranteed allocation of cpu/mem
- Limits : Max allocation of cpu/mem

There is no defaults set, one pod can use the max and starve other resources in the cluster, so setting minimum is
usually advised.

## DaemonSets

* DaemonSets are similar to ReplicaSets, however works differently. It schedules a pod requested in the daemonset in
  every node that's available. When the node is terminated, the pods are also removed.
* The spec is otherwise the same as ReplicaSet with a selector (replicas and strategy don't apply).

Exam Tip: Structure of daemonSet is very similar to `Deployment`, so create a deploy.yaml and then modify it to fit
the needs of DaemonSets removing the unnecessary fields that don't apply to `DaemonSet`

Good use-case to use DaemonSet: Monitoring solution that should have a pod in every namespace, logs viewer.

## Static Pods

* When kubelet is set up, manifest path is set up for pods - `--pod-manifest-path`. Sometimes its also set via
  `--config` and a kubeconfig.yaml file will indicate `staticConfigPath`.
* When KubeAPI server is not available, then creating pods via this method is an option.
* Once configured, a pod object configuration file can be dropped to the path, and kublet will periodically poll
  this path and apply changes to pod object automatically without running any commands.

Please note that only pods can be created by this option.

How to find if pod is static pod or owned by standard k8s orchestration?

- Look at the `ownerships` block. If the owner is marked as Node, then its static pod, if the owner indicated
  is ReplicaSet then it is not a static pod.

## Multiple Schedulers

A custom scheduler can be deployed using `KubernetesSchedulerConfiguration` object along with required RBAC.

On the pod, `schedulerName` can be set which indicates the source of schedule used by the specific pod instead of the
default scheduler

## PriotizedClass

- `kind: PrioritizedClass`
- Pods can be designated high-priority or low-priority with a priority value ranging from 1000 to 2 million.
- Kubernetes reserves `system-critical` class with 2M value for k8s critical nodes.
- `PreEmptionPolicy` dictates what should happen to workloads already scheduled on the node. By default, the policy is
  set to `PreemptionLowPriority` which means all the existing pods that don't have the same priority to be evicted.

## How k8s scheduler works?

- k8s has many scheduling plugins - each one checks a specific attribute in a pod to derive the scheduling decision.
- When pods are created, they end up in a scheduling queue.
- In the queue, pods are sorted by their priorityClass.
- Once sorted, k8s filters the nodes that can run the workloads, places the pod per the prioritized class.

## Admission Controllers

* Everytime a request is sent, request goes to API server, object is created and persisted in ETCD.
* When request is sent to API server, it goes through authentication process using certificates.
* kubeconfig file has certificates configured, checks if user is valid
* Then authorization process kicks in and checks if user is allowed to execute given operation (get, list, create).
* RBAC / Roles determine if user has authorization.

What are the caveats of RBAC that drives the need for admission controllers?

* RBAC helps check if a given user has authorization to do the action, however it cannot scale for complex asks.
* For example, it cannot check if pod uses specific labels / appCode. It cannot validate if pod spec uses images from
  allowlisted image registries. This is the value offered by admission controllers.
* There are many admission controllers - `NamespaceExists`, `DefaultStorageClass`, `EventRateLimit` etc.,

Kubelet -> Authentication -> Authorization -> Admission Controller -> Create Pod

There are many admission plugins enabled only when needed.

How to find which admission plugins are enabled - 

`kube-api-server -h | grep enable-admission-plugins`

Now go to `/etc/kubernetes/manifessts` and find the object configuration file for `kube-apiserver.yaml`. use the
`--enable-admission-plugins` and `--disable-admission-plugins` to toggle the list.

### Mutating and Validating admission controller

- Mutating admission controller will create a change which can be pre-requisite step.
- Validating admission controller is an evaluation step - it evaluates if a given condition is met.

* Mutating admission controller runs ahead of Validating admission controller.
* Admission controller can be customized.