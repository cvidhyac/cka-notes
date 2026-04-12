# Application Lifecycle Management

## Rollout

* Each rollout triggers creation of a new replicaset.

3 commands - 

`k rollout status deploy/mydeploy`
`k rollout history deploy/mydeploy`
`k rollout undo deploy/mydeploy --revision 3`


## Commands and Arguments in docker

### Why do we specify CMD in docker?
* docker is only designed to run a process that has its own start up.
* `sh` or `bash` is a shell that looks for a terminal, they are not processes by itself.
* When a CMD is specified it keeps the process alive.
* 2 ways to specify a CMD - either specify as a string or json array. However, when specified in json array format,
the first argument should be an executable. So specifying `CMD ["sleep", 5]` is the right syntax.
* CMD can be overridden in command line, however both command and argument need to be overridden. CMD does not allow
override for value itself. This is where entrypoint shines.
* ENTRYPOINT specifies a command without specifying a value. The value can be specified while running the image. This
provides more flexibility.
* In case, user forgets to supply a value, entrypoint may fail. This is why we specify a `CMD` after `ENTRYPOINT` to be
able to provide a default value when user forgets to specify it.
* Entrypoint can be overridden with command using the `--entrypoint` option in the image.

### Commands and Arguments in kubernetes pod

* Any `CMD` with args in the `docker run` command is specified via the `args:` field in pod spec.
* The `--entrypoint` instruction in docker run command corresponds to `command:` in pod spec


## InitContainers and sidecars

* In the latest version of kubernetes, we can reuse `initContainers` to specify that it should be a sidecar.
* By default, an initContainer does not restart hence its `restartPolicy: false`, however by specifying the 
`restartPolicy: true` then it will behave as a sidecar.

## Manual, Horizontal and Vertical Pod Autoscaling

### Manual Autoscaling
A k8s administrator edits the `kind: Deployment` to increase the resources to be available
for pods. These updates are not applied "in-place", the existing pod will be killed and new pods are provisioned 
with the updates. Depending on the production workload criticality, 

### Horizontal pod autoscaling (HPA)
Specify a specific CPU / Memory utilization condition when the pod should be 
scaled. When condition is met, k8s adds a pod with same amount of resources. This is identified by
`kind: HorizontalPodAutoscaler`

### Vertical Pod AutoScaler (VPA)
VPA is a new alpha feature and not stable yet, hence this is not a "built-in" feature.
The CRD for VPA has to be installed directly from k8s github repo. 

To visualize how VPA processes the autoscaling requests read - https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/

Since VPA is not built-in, there is no imperative command to create VPA, it needs to be created declarative.

VPA has 3 core components:
- VPA Recommender: Queries k8s metrics-server and publishes metrics
- VPA Updater - Evicts and recreates new pods based on inputs events from AdmissionController for pod deployment resource recommendations
- VPA Admission Controller - Invokes K8s API server to update `kind: Deployment` with updated resource recommendations.

There are 4 modes how VPA applies the updates to the pod as defined in `updatePolicy`:
- OFF: VPA recommender publishes the recommendations but VPA Updater and Admission Controller do not react to these.
- Initial: VPA Recommender events are only accounted for initial pod creation or the next time when deployment is
restarted and pod is recreated but not applied instantly.
- Recreate: Pods are evicted and recreated when resource threshold exceeds
- Auto: Behaves like "Recreate" and when "In-place" updates are possible then opts for this mode when supported.


```shell
k describe vpa my-vpa
Recommendations:
  Target:
    cpu: 1.5
```

Differences between VPA and HPA:

- Scaling method: VPA updates existing pods, HPA adds new pods
- Pod Behaviour: VPA restarts pod, so some disruption is possible. In HPA, additional pods are added so no disruption
- Handle traffic spikes? HPA is the winner here because it instantly adds more resources, VPA has to wait for pods
to be updated with new resources and it can be disruptive.
- Optimizes cost: No winner here, each one has a specific way of optimizing cost by optimizing existing resource
configuration or by killing idle-resources/pods
- Best for: VPA is best for DB's, ML workloads. HPA is best for webapps, microservices and stateless services