# Logging and Monitoring

To set up metrics server in kube-system manually use the slimmed down metrics server available in kubernetes releases.

## How are metrics collected in kubernetes?

* Each kubelet runs a process called C-Advisor or container advisor that sends the data from the pods
to the metrics-server. 
* There is ONE metrics server in each node, and this stores the data "In Memory".

## common metrics commands

`k top nodes` --> Top CPU and memory consumers in node
`k top pods` --> Metrics for CPU and memory at pod level

## check logs in k8s

`k logs -f pod_name container_name` --> -f = follow
