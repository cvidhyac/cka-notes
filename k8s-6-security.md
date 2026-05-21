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