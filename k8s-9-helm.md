# Helm for CKA

* Assume helm is already installed, no need to memorize package managers or curl commands how to install helm on k8s
  cluster.
* Use `helm help` to find information.

## How does helm perform rollback?

* It compares live state with revision and decides how to go back to the previous revision.
* When `helm rollback package_name` is requested, helm performs this rollback via 3-way strategic merge patch to
  rollback image version.

## Helm components

- Charts: The artifacts that should be installed together as a package
- Release: When a chart is applied to a k8s cluster, it creates a release
- Metadata: Changes to the release is produces state changes. These state changes are tracked in metadata, and helm
  saves the metadata as a k8s secret

## Helm Releases

`helm install release_name package_name`

The reason we specify a release name is to be able to manage two different entities or releases from the same package
possible.

`helm install release_1 bitnami/wordpress` and `helm install release_two bitnami/wordpress` can co-exist in same
namespace.

Similar to dockerhub, community supported helm charts can be downloaded from `artifacts hub` also called as
`artifacts.io`

## Helm templates

* Uses syntax `{{ .Values.placeholder_path }}`
* values.yaml file contains the placeholders and their values.
* Helm 2 uses `apiVersion: v1` and Helm 3 uses `apiVersion: v2`. Helm 3 also introduces `dependencies` to differentiate
  older charts from newer ones.
* Every chart must have its own version. This helps helm track changes to the chart itself.
* There are two types of charts - `type: application` and `type: library`. Application is a type of chart that can
be installed. Library is a utility that helps build other charts.

## Key commands

`helm repo add repository_name`

`helm repo list`

`helm install release_name package_name`

`helm search repo/hub package_name`

`helm rollback release_name revision_number`

`helm upgrade release_name package_name --version`

`helm install --set any_values_yaml_key=desired_value release_name package_name`

