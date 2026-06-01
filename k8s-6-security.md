# Secure Kubernetes

## Security primitives

- Control access to the API Server: Who can access the cluster and what can they do?
- Secure with TLS Certificates between each k8s components
- Add network policies

## Cluster Authentication

- Human users
- Applications that access different k8s resources

K8s does not manage authentication. K8s does not manager user access.

Service accounts can be created and managed on k8s.

All user access is managed by api-server. kube-apiserver authenticates the user using different authentication
mechanisms configured on API Server:

- static token files
- Certificates
- OAuth2 or other third party protocols.

* Static token files have hardcoded tokens and not recommended for production. It is a beginner concept.

## TLS Certificates basics

All k8s manifests files are located in - `/etc/kubernetes/manifests` directory. Each k8s component has its own yaml
file.

- There are three types of certs - server.crt/server.key, client.crt/cert.key and ca.crt/ca.key
- Certificates exist as key pairs. `server.key` is the private key and `server.crt` is the public key.
- A client is authenticated to the server using their client.crt, ca.crt/ca.key is the issuer or signing authority.

All Certificate related operations are handled by Kubernetes "Controller Manager" component.

- A CertificateSigningRequest is sent by the requester to the issuer.
- Issuer 'Approves' the certificate signing request, and a certificate is issued.

### How to create a k8s CertificateSigningRequest?

If a server.csr is available, create a base64 copy of this one -

`cat server.csr | base64 | tr -d "\n"`

Then paste this into the csr.yaml file -

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser # example
spec:
  # This is an encoded CSR. Change this to the base64-encoded contents of myuser.csr
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
    - client auth
```

Once this is applied, query it by running `k get csr`, its default status will be `Pending`

To approve/deny this csr, run:

`k certificate approve myuser`
`k certificate deny myuser`

Once it is approved or denied, it can be queried via `k get certificate`

## kubeconfig

kubeconfig has 3 sections - clusters, context and users.

- cluster define the k8s cluster configuration
- users section defines the name of the user and their auth
- context marries cluster and user to provide the context `cluster@user`

important commands to know:

`k config view`
`k config use-context user@prod` --> to change context

### how to switch to a default namespace in kubeconfig?

specify the namespace in the contexts section -

```yaml
contexts:
  - context:
      cluster: production
      user: myuser
      namespace: finance
      name: production@myuser
```

### how to specify certificate in kubeconfig?

Two options:

- `certificate-authority` field accepts a path to the `.crt` file
- `certificate-authority-data` field accepts a base64 encoded content of the certificate in cases where path cannot
  be specified.

## API Groups

- API server can be accessed at: curl https://kube-master:6443/api/version
- The kube-api server is default authenticated and can be accessed by providing `--cert` and `--key` parameters.
- To avoid providing the cert and key parameters everytime, we can set up a `kubectl proxy`. This uses the certificate
  and key credentials configured in `kubeconfig` and automatically helps forward the request to the API server and allow
  us to access API without providing additional auth.
- However, note that `kubectl proxy != kube proxy`. The kubectl proxy is a HTTP proxy.
- k8s has two groups of API: Core API groups and Named API groups. All resources are defined under named api group.
  Core api group is where all the resources are collectively grouped (`storage.k8s.io`)

## Defining authorization

k8s has 4 modes of authorization: Node, ABAC, RBAC, Webhook

Node: `system:node` access for elevated admin access and k8s internal access.
ABAC: Attribute based access control. Defines a policy per user with actions that are allowed.
RBAC: Role based access control. Defines a role instead of per-user definition. This is the standard mode of
authorization.

Webhook: Manages authorization externally - outsourced to a third party tool like Open Policy Agent (OPA).

- For example, OPA is a third party tool used for managing auth.
- The kube api server makes an API call to OPA with the user details and access requested.
- Based on the response returned by OPA, user is granted or denied access.

The default authorization mode if not specified in manifests in kube-apiserver (
/etc/kubernetes/manifests/kube-apiserver.yaml)
When not specified the `--authorization-mode` is set to `Always Allow`. Multiple authorization modes can be configured.

Multiple authorization modes are evaluated in the order - `--authorization-mode=Node,RBAC,Webhook`

### Short note on RBAC

- RBAC has two parts - define a role, then a role binding for the user and the roleRef.
- `Role` is created first, followed by `RoleBinding` objects. Role indicates `apiGroups`, `resources` and `verbs`
- `RoleBinding` defines the user, name and role Ref. An `apiGroup` is specified in `RoleBinding` specifying that both
  user and roleRef belongs to RBAC Authorization api group `rbac.authorization.k8s.io`.

### Checking Access

Run `k auth can-i <action> --as <user>` command to check if a specific action is allowed.

For example:
`k auth can-i create deployments --as johndoe`

## Cluster Role and Cluster Role Bindings

- Not associated with any namespace
- Use the CLI command to create them, it automatically finds and creates the `apiGroups` for the yaml.
  `k create clusterrole --help`
- `k create clusterrolebinding --help`

## Service Accounts

- Service Account is used for authenticating to a given resource, it auto mounts by default.
- A default service account by name `default` is created in the namespace and is default attached to pods.
- The service account is mounted as a "projected volume". A projected volume is a dynamic directory. Token is available
  as a file.
- kubelet automatically rotates the token when it expires.
- `k create token service_account_name` creates a token that can be used on a ci pipeline

## Image Registry Secrets

`k create secret docker-registry <secret_name> --docker-username --docker-password --docker-server --docker-email`

Then specify the parameter `imagePullSecrets` with the secret name created above in pod template to reference the
credential to be used for pulling image from private image registry.

## Docker Security

- By default, docker runs on root user, to change this behaviour, pass the `--user` with command or specify `USER` in
the yaml file.
- Default docker run does not allow all capabilities on the OS. To add a capability use `--cap-add` or to drop feature
use `--cap-drop`

## K8s Security

- similar to docker the capabilities can be configured on the pod template. It can be defined either on the container
level or pod level. When defined on pod level, it is applicable to all container running in the pod. When defined on
container level, it is more fine grained.
- When a specific security configuration is defined at both pod level and container level, container level takes 
precedence.

```yaml
securityContext:
  runAsUser: 1000
  capabilities:
    add: ["MAC_ADMIIN"]
```

Please note that capabilities are only allowed to be defined at container level and not available at pod level.

To find which user is running given process run - `ps aux`

## Network policy basics

- Ingress: Incoming Traffic
- Egress: Outgoing traffic from current pod

For example, if web traffic is sent by user on port 80, API server is hosted on port 5000, API invokes DB on port 3306
following would be the rules:

- Web ingress on port 90
- Web egress to port 5000
- API ingress on port 5000
- API egress to port 3306
- DB ingress to port 3306

In all the cases, the port used for returning response to user doesn't matter.

### Developing Network policies

- The direction of arrow/request determines ingress or egress (inward arrow is ingress, outward arrow is egress)
- podSelector is used to narrow down the pod that should be protected

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              name: api-pod
          namespaceSelector: ## allow only api-pod in prod namespace to reach the db
            matchLabels:
              kubernetes.io/metadata.name: prod
      ports:
        - protocol: TCP
          port: 3306
  egress:
    - to:
        - ipBlock:
            cidr: 192.168.5.10/32 # Outgoing traffic to DB Back up server on port 80
      ports:
        - protocol: TCP
          port: 80
```

## Custom Resource Definition (CRD)

- A custom resource is a k8s controller. It monitors the changes in the cluster then executes changes by invoking 
an API or external call to sync the state of the resource.

For example, how does a k8s deployment get created?

- A yaml file excecution command `k create -f deployment.yaml` is executed in CLI.
- This causes an entry to be added to the etcd database in cluster
- The k8s `deployment` controller watches for the entry that creates a new `kind: Deployment`
- It then executes a change by invoking the k8s API to create a new deployment object in the cluster.

Same process for any CLI command, under the hood there is interaction with the etcd database AND/OR additional
change executions to create and manage the k8s objects. CRD helps manage the state of objects in k8s cluster.

- CRD (Custom Resource Definition) is the spec for the API
- Custom Resource is the actual invocation that uses the CRD to create the object
- When k8s command is executed, entry is added to etcd, then k8s controller executes change to create the `CustomResource`
following the CRD definition.
- CRD and CustomResource can be either cluster scoped or namespace scoped.


### Developing Custom Controllers

- Listens to specific k8s objects. Written in some programming language such as golang.
- It watches for changes, makes API calls to create the objects in system.
- Packaged as an image and run as a pod in k8s cluster.

### Operator Framework

- An operator mimics human interaction in cluster - monitors cluster, installs, takes backup, patches, restoration tasks.
- Custom controller has the logic to work with the CRD.
