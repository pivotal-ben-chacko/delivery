## Namespace Provisioner (Controller Mode)

[Namespace Provisioner Documentation (TAP 1.7)](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/namespace-provisioner-use-case3.html)

Namespace provisioner will create a secret called registries-credentials that will contain any registry secrets that have been exported to all namespaces using the following command:

tanzu secret registry add harbor-credentials --server harbor.skynetsystems.io --username admin --password changeme --export-to-all-namespaces --yes --namespace tap-install

Use `tanzu secret registry list -A` to see all secrets that have been exported to all namespaces.

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

For private repositories create a secret ref that points to the git credentials, see: [Doc](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.6/tap/namespace-provisioner-use-case3.html#git-private)

Once all required credentials are exported, label your namespace for the controller to provision it: `kubectl label namespaces YOUR-NEW-DEVELOPER-NAMESPACE apps.tanzu.vmware.com/tap-ns=""`

In order to select and apply the appropriate Tekton Pipeline and Scan policy, add the following labels to the namespace:

```
  kubectl label namespaces YOUR-NEW-DEVELOPER-NAMESPACE tap.tanzu.vmware.com/pipeline=java
  kubectl label namespaces YOUR-NEW-DEVELOPER-NAMESPACE tap.tanzu.vmware.com/scanpolicy=lax
```

YTT logic inserted at the very beginning of each file in the **git-ops/params** directory will or will not get applied based on the profile running in the cluster and the label assigned to the namespace.

In the case above: 
  - the **java** pipeline will be selected and applied if profile is (full, iterate, build) and the label **tap.tanzu.vmware.com/pipeline: java** is assigned to the namespace
  - the **lax** scan policy will be selected and applied if profile is (full, build) and the label **tap.tanzu.vmware.com/scanpolicy: lax** is assigned to the namespace


## Git authentication for workloads and supply chain

Set up namespace provisioner to automatically add git credentials to the namespace and update the default service account to use it. The actual secret will be stored in the namespace **tap-install** called **workload-git-auth**, and referenced by a secret template stored at the git repo location definded in **additional sources** section in tap-values.

1. Add the git secret to tap-values

```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: workload-git-auth
  namespace: tap-install
type: Opaque
stringData:
  content.yaml: |
    git:
      #! For HTTP Auth. Recommend using https:// for the git server.
      host: GIT-SERVER
      username: GIT-USERNAME
      password: GIT-PASSWORD
      caFile: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
EOF

2. Add the following secret template to git repo in path **git-ops/params**
```bash
#@ load("@ytt:data", "data")
---
apiVersion: v1
kind: Secret
metadata:
 name: git
 annotations:
   tekton.dev/git-0: #@ data.values.imported.git.host
type: kubernetes.io/basic-auth
stringData:
 username: #@ data.values.imported.git.username
 password: #@ data.values.imported.git.token
 caFile: #@ data.values.imported.git.caFile
```

3. Now update tap-values file with the following info (the new values go under **import_data_values_secrets**):
```
namespace_provisioner:
  controller: true
  additional_sources:
  - git:
      ref: origin/main
      subPath: git-ops/params
      url: https://github.com/pivotal-ben-chacko/delivery.git
  import_data_values_secrets:
  - name: workload-git-auth
    namespace: tap-install
    create_export: true
  default_parameters:
    supply_chain_service_account:
      secrets:
      - git
```
