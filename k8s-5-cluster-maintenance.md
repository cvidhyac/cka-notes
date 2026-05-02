## Cluster Maintenance

OS upgrades - K8s needs a strategy for this one. 

Before applying upgrades, the workloads in the current node are "drained". It means the pods are terminated from 
current node and created on another one.

The node is also "Cordon" ed. It means the node is marked unschedulable. After upgrade, an "UnCordon" action is 
done.

Cordon does not terminate existing workloads, it only stops new workloads from being added to the node.

### Command examples to use during cluster upgrade

**Make node unschedulable**

`k cordon node01`

**Drain workloads from node**

`k drain node01`

### Cluster Upgrade Sequence

* Upgrade by every minor version
* Upgrade kubeadm first, then kubelet then other components
* Upgrade every worker node by cordoning it, draining it then running upgrade on kubeadm followed by kubelet.

#### Upgrade kubeadm

```shell
apt-get upgrade -y kubeadm=1.12.0
kubeadm upgrade apply v1.12.0
```

#### Upgrade kubelet on master node (if kubelet exists in master node)

```shell
apt-get upgrade -y kubelet=1.12.0
systemctl restart kubelet
```

At this point, worker nodes will still be on lower minor version. They need to be cordoned and drained.

```shell
k cordon worker01
k drain worker01

apt-get upgrade -y kubeadm=1.12.0
kubeadm upgrade apply 1.12.0

kubeadm upgrade node config --kubelet-version v1.12.0
apt-get upgrade -y kubelet=1.12.0
systemctl restart kubelet

k uncordon worker01
```

Find the latest version of kubeadm available to upgrade - 

```shell
kubeadm upgrade plan
# Look for remote version or grep it
```

Follow the doc to upgrade kubeadm and kubelet. Always remember to `cordon` and `drain` the `controlplane` or worker node
before upgrade.

Before the upgrade, the keyring file may need to be updated to the version that needs to be updated. To find the 
command to update the keyring file see this doc - 
https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/

For detailed upgrade commands see -
https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

ssh to the worker node to update it, for example `ssh node01`

after upgrade `uncordon` the node.


### Back up and Restore Strategies

- Get all pods and resources and extract them to a yaml file 

`k get all --all-namespaces -oyaml > all-deploy-svc.yaml`

- Many other resources may need to be considered, we usually use a tool like ARK to take backups.

How to back up etcd?
- etcd stores details about the cluster, we need to backup etcd server itself.
- `etcdctl snapshot save snapshot.db`
- `etcdctl snapshot status`

To restore from backup, first stop `kube-apiserver`

- `systemctl stop kube-apiserver`
- `etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup`

When etcd is restored from backup, a new data directory is created and added as new member so that it doesn't override
an existing member.

```shell
systemctl system reload
systemctl restart etcd
systemctl restart kube-apiserver
```
Read this doc to back up and restore etcd - 

https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md

