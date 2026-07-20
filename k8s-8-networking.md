# Networking Concepts

## Networking Pre-requisites

* Switching and routing concepts
    * Switches
    * Routing
    * Default Gateways
* DNS introduction
    * CoreDNS intro
    * DNS configs on linux
* Network namespaces

## Visualizing a switch and route

- Two devices A and B need to talk to each other. We add a switch in between.
- Run `ip link` command on Device A to find out its interface name (`eth0`)
- Run `ip link` command on Device B to find out its interface name (`eth0`)
- Two devices with same interface name can be connected through a switch.
- First, add Device A to switch: `ip addr add 192.168.1.10/24 dev eth0` --> device A connected to switch
- Then, add Device B to switch - `ip addr add 192.168.2.35/24 dev eth0` --> device B is now connected to switch

![how-switch-and-route-works](attachments/how-switch-and-route-works.png)

When two different networks need to send packets to each other, lets say Device A in network 1 needs to ping
Device C in Network 2, a route is required from their existing switches.

Run command `ip route` on each system to check if it is connected to any route

Device A can talk to Device C through route one. Create this route by running a command -

Add this routing table rule on System A
`ip route add 192.168.2.0/24 via 192.168.1.1` --> Reach IP network 2.0 through the door 1.1

Now Device C needs to send response back to Device A, so it needs a routing rule as well -

Add this routing table rule in System D
`ip route add 192.168.1.0/24 via 192.168.2.1` --> Reach IP network 1.0 through door 2.1

### How would devices connect to internet?

- First the network that offers connection to internet is connected to the route and it has an IP.
- Then, routing table rule is added to each device that wants to connect to internet.

To allow devices in network 2.0 to access internet add routing table rule for devices C and D :
`ip route add 172.217.194.0 via 192.168.2.1`

To allow devices in 1.0 network to access internet, another routing table rule is needed in each device A and B
`ip route add 172.217.194.0 via 192.168.1.1`

## When do you need a Default Gateway?

- When there are multiple routers, then add a gateway to reduce the number of entries in each systems in particular
  network.

`ip route add default via 192.168.2.1`

## Set up linux host as a router

- In linux hosts, packets from one interface is not automatically forwarded to another.
- For example, `eth0` is not forwarded to `eth1`. This is for security reasons. Let's say eth0 is on private network and
  eth1 is on public network, we do not want folks in public network to be able to easily send messages to private
  network.

### Important note

- Changes made via commands `ip addr` or `ip route` is only valid until hosts are restarted.
- To persist these changes, it should be made via `/etc/network-interfaces` files

## DNS, NameResolution and NameServers

### Hosts file

- A server can ping another server by either an IP or a name.
- Name is defined in `/etc/hosts` file if a server does not have to be discoverable by other devices in network.
    - An entry in `/etc/hosts` file looks like this: `192.168.2.15 db`. Now we can ping the named host as `ping db`.
    - When using hosts file to define entries, entry needs to be made per device in the network. each server has its own
      hosts file.
    - This scales well for small number of servers, however as the number of server grows, it becomes too tedious to
      manage name changes.

The process of resolving from name -> IP is called as "Name resolution"

### DNS Server

- As the number of servers grow, it becomes too tedious to manage IP and their names. Hence we add another linux host
  with its job to act as a DNS server. This DNS server would now become the central repository of all the IP and their
  names.

```text
192.168.2.15 test
192.168.3.33 db
192.168.5.23 api-server
```

- Everytime any server wants to look up another server's name, it always checks the DNS server.
- The DNS server here is called a "nameserver" and is configured in every server. The location of the nameserver is
  configured in `/etc/resolv.conf`

A `resolv.conf` entry looks like this:

```shell
nameserver 192.168.6.1
nameserver 8.8.8.8
```

Now, the `resolv.conf` can have entries for many nameservers. They are consulted in the order. The nameserver `8.8.8.8`
is maintained by google, and it has the names of all publicly discoverable IP and names listed in it.

Hence, when the server pings, `ping www.facebook.com`, the request is forwarded to the `8.8.8.8` server since the
private nameserver `192.168.6.1` has no entry configured for this one.

### Controlling the order of name resolution

Even if a DNS nameserver exists, the server can still hold additional entries in `/etc/hosts` file. The order of
resolution is defined in `/etc/nsswitch.conf` file.

```shell
nameserver: files dns
```

By default, this file defines `files` first followed by `dns`. This means local `/etc/hosts` file is consulted first
then followed by the external nameserver. The order can be swapped if needed in this file.

### Configuring domain names

Say there is a domain name called `apps.myorg.com`, and we run `ping apps`, it fails because of name resolution error
because there is no direct entry for the domain name `apps.myorg.com` in the /etc/hosts file. The DNS Server however
has the entry already however the name configured is different

```shell
192.168.4.34 apps.myorg.com
192.168.6.23 web.myorg.com
```

Why didn't the ping automatically append the domain name?
This is because the default setting to append domain name for internal sites has to be configured
in `/etc/resolv.conf` file.

```shell
nameserver 192.168.6.1
nameserver 8.8.8.8
search myorg.com # This entry facilitates the search by appending myorg.com to all pings for names
```

### Record Types

How are records stored in DNS server?

- `A` RECORD: An name mapped to IP is called `A` record
- `AAAA`RECORD: Quad A records are names mapped to IPV6
- `CNAME` RECORD: One name mapped to the other.

### Other commands

`nslookup` : `nslookup` only queries DNS server, it does not query `/etc/hosts` file.
`dig` : similar to `nslookup` this also only queries dns nameserver.

## CoreDNS

- Download the executable using curl -O and then run `tar -xzf` to extract the `.tgz` file.
- Run the executable, this should start a DNS server that listens on port 53.
- CoreDNS uses a `Corefile` that configures where `/etc/hosts` is located and any entries not available in hosts file
  is then configured to be forwarded to the `/etc/resolv.conf` file in the `Corefile` itself.
- CoreDNS also supports configuring entries through plugins.

## Network namespaces - Introduction

When we create a container we do not want it to access process of other containers. This is where network namespaces
is helpful.

A network namespace helps isolation (virtual isolation) for each container running in a host.

Each host has a routing table and ARP table. When container is created we create a network namespace for it.
Each container has its own interface, routing table and ARP table.

### Create network ns

`ip netns add red`

### List network ns

`ip netns`

### List interfaces within network ns

Prefix the commands with `netns exec <netns_name>` to query the network namespace

`ip netns exec <netns_name> ip link`

`ip netns exec <netns_name> ip addr add`

The syntactic sugar is `-n` equivalent of `netns exect` - `ip -n red link`, `ip -n blue route`

### Establish connectivity between two network namespaces

Let's say we have two network namespaces `red` and `blue` within a host, and we need to establish connectivity between
these two network namespaces.

- Create a virtual ethernet pair for each of the network namespaces and connect the two ends
  `ip link add veth-red type veth peer name veth-blue`

Now attach the virtual interface to each of the network namespaces respectively - veth-red attached to red and veth-blue
attached to network namespace blue.

`ip link set veth-red netns red`
`ip link set veth-blue netns blue`

Now run the `ip addr add` command to connect the two network namespaces to a switch.

`ip -n red addr add 192.168.15.1 dev veth-red`
`ip -n blue addr add 192.168.15.2 dev veth-blue`

Now bring the links up -

`ip -n red link set veth-red up`
`ip -n blue link set veth-blue up`

Test connectivity by pinging them -

`ip -n red ping 192.168.15.2` should now return an answer
`ip -n blue ping 192.168.15.1` should also return an answer

Querying ARP table should now have a mac address entry -

`ip -n red arp`
`ip -n blue arp`

## Virtual Network And Bridge

To create an internal bridge network, we add a new interface to the host using `ip link add` command
`ip link add v-net-0 type bridge`

To the host this is another interface similar to other interface. By default when a new interface is created, it is
always down. We need to bring it up.

`ip link set dev v-net-0 up`

Now we need to connect ALL namespaces to bridge network.

So, this link doesn't make sense anymore.

`ip link -n red del veth-red` --> Other end automatically gets deleted

Now create a link to connect veth-red to bridge.

`ip link -n red veth-red type veth peer name veth-red-br`

`ip link -n blue veth-blue type veth peer name veth-blue-br`

Now connect this back to red network namespace and also attach to the bridge.

`ip link set veth-red netns red`
`ip link set veth-red-br master v-net-0`

`ip link set veth-blue netns blue`
`ip link set veth-blue-br master v-net-0`

Now add the addresses to attach an IP to these network namespaces

`ip -n red addr add 192.168.15.1 dev veth-red`
`ip -n blue addr add 192.168.15.2 dev veth-blue`

Now bring this up,

`ip -n red link set veth-red up`
`ip -n blue link set veth-blue up`

## Docker Networking

3 types:

- None: Container is not mapped to any network, host cannot be accessed by the host.
- Host: Container and host share the same port. If the container runs a process on port 80, it is accessible from host
  on the same port.
- Bridge: Default in docker when container is run, docker manages all ip table entries.

In Bridge mode, the container is accessible within the host, however external access is not allowed. To enable external
access, "port mapping" should be used with `-p 8080:80`

How to check what is the current network list in docker?
`docker network ls`

## Configuring CNI

CNI = Container Network Interface, k8s compliant specification

CNI plugins are the network solution providers

* k8s supports many network plugins such as flannel, weave, default linux bridge, vmware nsx-t etc.,
* The CNI plugins are installed in `/opt/cni/bin` directory. The bin directory contains the executables.
* The network configurations for the plugins are at : `/etc/cni/net.d` directory. The configuration files have a
  `.conflist` suffix
* The conf files have a specific structure as directed by the CNI specification. It has many sections and indicates
  the network plugin configuration.

A basic plugin does the following:

- Create veth pair
- Attach veth pair
- Assign IP Addresses
- Bring up the interface
- Delete veth pair

## IP Address Management (IPAM)

* k8s does not care how IP Addresses are managed. It only requires that duplicate IP Addresses are not assigned.
* The easiest way to do this is by storing the details of the IPs in a file, and a script that invokes it.
* The script would invoke host-local / dhcp plugin that would invoke the plugin that assigns IP and stores the
  assigned IP address to a file.

### How to find which CNI plugin is installed on cluster?

- `cd /etc/cni/net.d`
- `ls` to find which conflist is present (minor-cniplugin.conflist, e.g. 10-flannel.conflist)
- cat the conflist and find the `type`. File name also indicates it

Note:
Flannel does not care about how containers are networked to the host. It only cares about how traffic is transported
between hosts. Flannel is focussed on networking and does not support network policies. It recommends Calico for
network policies.

### How to ping the pod curl works?

`k get pods -owide` --> find the IP of the destination_pod to check connectivity
`k exec -it source_pod -- curl -m 5 destination_pod_ip`

### How to delete a CNI?

- Delete the DaemonSet
- Delete the configmap
- Delete the configuration file

## How do pods communicate to each other in same node or same cluster?

- They communicate to each other through Services. And default `svc` is of type `ClusterIP`
- This works until both pods are on the same node.
- When k8s services are created, they are accessible from all pods on the cluster irrespective of what nodes they are
  on.
- `svc` is not bound to specific node, it is assigned a ClusterIP. Service is only accessible from all nodes in same
  cluster. Service is hosted across the cluster

## How would pods communicate to each other on different cluster?

- When pods should be accessible to a pod in another cluster, it should use a `NodePort` service.
- `NodePort` is similar to `ClusterIP`, in addition it exposes a port for external users to access the service.

## How do services get IP Addresses, and how are they accessible to external users?

- In a 3-node cluster, there is a kubelet in every node. Kubelet checks kube-apiserver to monitor any changes to 
cluster.
- When pods are requested to be created, kubelet creates the pods. It then invokes the CNI plugin to get the pod
networking configured.
- In each node, there is also a `kube-proxy` component. This also checks cluster activity and changes through
kube-apiserver.
- When services are requested to be created for pods, then kube-proxy springs into action.
- The service is assigned an IP from a pre-configured range (check kube-controller-manager.yaml in `/etc/kubernetes/manifests`
- Once Service has an IP assigned, `kube-proxy` starts creating forwarding rules from serviceIP:port to the pod's IP.

How are these forwarding rules created?
- Kube proxy supports many modes - userspace, ipvs and iptables. Default is iptables. The default mode how kube-proxy
is configured can be changed via a commandline parameter or by checking its logs.
- Everytime a service is created or deleted, the associated forwarding rules are also added or deleted.
- kube-proxy is managed through a `DaemonSet`

To find the default IP range of the given node, run `ip link`, find the default interface. Then query,
`ip addr show default_interface_name`


### How to know what type of kube-proxy is being used?
Look in kube-proxy pod logs

## k8s DNS

- CoreDNS is recommended for recent versions of k8s. Internally it uses a service by name `kube-dns`.
- k8s replaces dashes against an IP address containing dots, then adds the entries.
- entries for each pod is located in `/etc/resolv.conf`
- To find clusterIP, run the commands with `-owide`

## Ingress

- Ingress and Services are not the same. Ingress is deployed in front of load balancer to route traffic to different
subdomains.
- Ingress is deployed as a controller. Ingress resources are created to manage the routing configuration with
different paths to route to the right application.
- Ingress can be created two ways:
  - create by path against same domain name by just separating backend
  - create by different http hostnames and in addition separating the backend path as well.
- Ingress Resources contain the rules.

Limitation of Ingress:
- Shared
- No support for many multi-tenancy features such as rate limiting, traffic splitting etc.,

Gateway API is recommended for enterprise multi-tenancy use cases.

## Gateway API

Gateway is the next-gen Ingress solution in k8s that address many multi-tenancy concerns.

This architecture allows for 3 main objects, each managed by different set of persona.

* Gateway Class: Managed by Infrastructure / Cloud Providers
* Gateway Operators: Managed by Cluster Operators (k8s/openshift admins)
* HTTP Route configurations: Managed by Application Development teams

When configuring httproute, remember to check which namespace the gateway is present and add `namespace` in addition
to `name` against the parentRefs.

