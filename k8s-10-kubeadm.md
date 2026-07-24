## Installing kubeadm

Read and follow - https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### Tips to install

- set up ipv4 forwarding first. Verify it worked.

```shell
> cat <<EOF sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward=1
EOF

> sudo sysctl --system
> sysctl net.ipv4.net_forward
net.ipv4.ip_forward=1
```

- Run `sudo apt-get update`
- check `systemctl status containerd/crio`
- check linux version in `/etc/*release` or `uname -a`

### Installing tips

- Note the version asked in question. Go step by step.
- Verify IP forwarding is set up
- Follow doc and set up keyring
- When it is time to install, pin the versions first
  `sudo apt-get install -y kubelet=version kubeadm=version kublet=version`
- Mark it on hold

Verify installation was successful by verifying each of the versions

```shell
kubeadm version
kubectl --version
kubelet version
```

The next step would be to repeat this on the node.

### Setup Node networking

Run `kubeadm init` on controlplane with the right parameters.

Find the ip of the controlplane looking at `eth0` interface running `ip addr`

Check nodes already joined by checking `k get nodes`. If nodes are not seen, then they need to join the controlplane.

When kubeadm is initialized successfully, a join command will be printed at the end. If you forget to copy this token,
it needs to be created again.

`kubeadm token create --print-join-command`

SSH into the node, then paste the token in the node using the output of previous command. If join was successful, the
node would indicate that this can now be checked by running `k get nodes` on controlplane.

### Install Flannel

use the flannel github repo, you can use the helm chart way to install it. Before you do this, find out the podCIDR
range.

To find pod CIDR range, go to `/etc/kubernets/kube-controller-manager.yaml` file and look at the cluster-ip-range.
Provide this value in the helm installation command. This sets it up with right IPs.

If you forget to set it up in helm install, it needs to be fixed by editing the flannel config map in `kube-flannel`
namespace.

Then edit the deployment to set `--iface=eth0` parameter in the container args section in pod template
along with other ip subnet params.