## Namespace Provisioner (Controller Mode)

Namespace provisioner will create a secret called registries-credentials that will contain any registry secrets that have been exported using the following command:

tanzu secret registry add harbor-credentials --server harbor.skynetsystems.io --username admin --password changeme --export-to-all-namespaces --yes --namespace tap-install


tap-values file will need to be updated with the following parameters:

```
namespace_provisioner:
  controller: true
  controller_resources:
    resources:
      limits:
        cpu: 500m
        memory: 100Mi
      requests:
        cpu: 100m
        memory: 20Mi
  namespace_selector:
    matchExpressions:
    - key: apps.tanzu.vmware.com/tap-ns
      operator: Exists
  additional_sources:
  - git:
      ref: origin/main
      subPath: git-ops/params
      url: https://github.com/pivotal-ben-chacko/delivery.git
      secretRef:  # secretRef section is only needed if connecting to a Private Git repo
        name: git-auth
        namespace: tap-install
        create_export: true
  parameter_prefixes:
  - tap.tanzu.vmware.com
```
