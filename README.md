# RHACM Cluster Policies with OCI Artifacts

This repo defines Red Hat Advanced Cluster Management (ACM) policies that Argo CD deploys from an OCI artifact in Quay, so the cluster doesn't need to connect to a Git repository.

```text
Repo -> OCI artifact in Quay -> Argo CD repo-server pulls and unpacks it
-> Kustomize + PolicyGenerator render it -> Argo CD applies the policies to the hub
-> ACM propagates them to clusters labeled base-config=true
```

## Prerequisites

- An OpenShift hub cluster with ACM and OpenShift GitOps installed
- `local-cluster` registered as a managed cluster
- `redhat-operators` available in `openshift-marketplace`
- `oc`, `oras`, and a private Quay repository

## 1. Enable PolicyGenerator in Argo CD

```bash
IMAGE=$(oc get deploy multicluster-operators-hub-subscription \
  -n open-cluster-management \
  -o jsonpath='{.spec.template.spec.containers[0].image}')
[ -n "$IMAGE" ] && echo "OK: $IMAGE" || echo "IMAGE IS EMPTY - stop here"
```

```bash
cat > argocd-policygen-patch.yaml <<EOF
spec:
  kustomizeBuildOptions: --enable-alpha-plugins
  repo:
    env:
      - name: KUSTOMIZE_PLUGIN_HOME
        value: /etc/kustomize/plugin
    initContainers:
      - name: policy-generator-install
        image: ${IMAGE}
        command: ["/bin/bash"]
        args:
          - -c
          - cp /policy-generator/PolicyGenerator-not-fips-compliant /policy-generator-tmp/PolicyGenerator
        volumeMounts:
          - name: policy-generator
            mountPath: /policy-generator-tmp
    volumeMounts:
      - name: policy-generator
        mountPath: /etc/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator
    volumes:
      - name: policy-generator
        emptyDir: {}
EOF

oc patch argocd openshift-gitops -n openshift-gitops \
  --type=merge --patch-file argocd-policygen-patch.yaml
oc -n openshift-gitops rollout status deploy/openshift-gitops-repo-server
```

## 2. Prepare the hub

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: openshift-gitops-policy-admin
rules:
  - apiGroups: [policy.open-cluster-management.io]
    resources: [policies, configurationpolicies, certificatepolicies, operatorpolicies, policysets, placementbindings]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [apps.open-cluster-management.io]
    resources: [placementrules]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [cluster.open-cluster-management.io]
    resources: [placements, placements/status, placementdecisions, placementdecisions/status]
    verbs: [get, list, watch, create, update, patch, delete]
  - apiGroups: [""]
    resources: [namespaces]
    verbs: [get, list, watch, create, update, patch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openshift-gitops-policy-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-gitops-policy-admin
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
---
apiVersion: v1
kind: Namespace
metadata:
  name: acm-policies-openshift-gitops
---
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: global
  namespace: acm-policies-openshift-gitops
spec:
  clusterSet: global
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-compliance
  labels:
    openshift.io/cluster-monitoring: "true"
    pod-security.kubernetes.io/enforce: privileged
    security.openshift.io/scc.podSecurityLabelSync: "false"
```

## 3. Give OpenShift user access to the Argo CD UI

```bash
oc adm groups new cluster-admins admin
```

## 4. Set up Quay

1. Create an empty, private repository named `rhacm-cluster-policies`.
2. Create a robot account named `rhacm_policies_push` with **Write** access to the repository.
3. Create a robot account named `rhacm_policies_pull` with **Read** access to the repository.

## 5. Build and push the OCI artifact

```bash
mkdir -p ~/.config/oras

COPYFILE_DISABLE=1 tar --no-xattrs --exclude '._*' --exclude '.DS_Store' \
  -czf acm-policies.tgz base

oras login quay.io -u 'adelledo+rhacm_policies_push' \
  --registry-config ~/.config/oras/auth.json

oras push --registry-config ~/.config/oras/auth.json \
  quay.io/adelledo/rhacm-cluster-policies:1.0.0 \
  acm-policies.tgz:application/vnd.oci.image.layer.v1.tar+gzip
```
The artifact has no base image. It's a single-layer package of files that Argo CD downloads and unpacks.

## 6. Connect Argo CD to Quay and create the ApplicationSet

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rhacm-policies-oci
  namespace: openshift-gitops
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: oci
  url: oci://quay.io/adelledo/rhacm-cluster-policies
  username: adelledo+rhacm_policies_pull
  password: <pull-robot-token>
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: acm-policies
  namespace: openshift-gitops
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - list:
        elements:
          - name: base
            path: base
  template:
    metadata:
      name: 'acm-policies-{{.name}}'
    spec:
      project: default
      source:
        repoURL: oci://quay.io/<quay-user>/rhacm-cluster-policies
        targetRevision: 1.0.0
        path: '{{.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: acm-policies-openshift-gitops
```

## 7. Sync the Application

```bash
oc -n openshift-gitops patch application acm-policies-base --type=merge -p \
  '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"syncStrategy":{"apply":{}}}}}'
```

## 8. Apply the cluster label in ACM

```bash
oc -n local-cluster get policy -w
```

## Update the image to 1.0.1 tag

1. Edit files under `base/`.
2. Rebuild the archive and push it with a new tag
    ```
    oras push --registry-config ~/.config/oras/auth.json \
    quay.io/adelledo/rhacm-cluster-policies:1.0.1 \
    acm-policies.tgz:application/vnd.oci.image.layer.v1.tar+gzip
    ```
3. Update the ApplicationSet:

   ```bash
   oc -n openshift-gitops patch applicationset acm-policies --type=merge -p \
     '{"spec":{"template":{"spec":{"source":{"targetRevision":"1.0.1"}}}}}'
   ```

## 9. track `latest` and update AppSet
 
This avoids patching the ApplicationSet for every release.
 
1. Switched ApplicationSet to 'latest'
 
2. Push new release to both new version tag and `latest`.
 
```bash
oras push --registry-config ~/.config/oras/auth.json \
  quay.io/adelledo/rhacm-cluster-policies:1.0.2,latest \
  acm-policies.tgz:application/vnd.oci.image.layer.v1.tar+gzip
```
 
3. Applied automated sync
 
```bash
oc -n openshift-gitops patch applicationset acm-policies --type=merge -p \
  '{"spec":{"template":{"spec":{"syncPolicy":{"automated":{"selfHeal":true}}}}}}'
```
 