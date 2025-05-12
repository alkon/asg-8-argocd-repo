### Home Assignmnet - Argo CD - GitOps in Action
#### Task 1: Setting up Argo CD
`a.` Install Argo CD
``` bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

`b.` Kubernetes Cluster Setup
- Expose Argo CD
``` bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
```
- Retrieve admin password (user: `admin`)
```bash
   kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```
- Login to Argo CD CLI
```bash
   argocd login localhost:8080 --username admin --password <admin-password> --insecure
```
---
#### Task 2: Deploy a simple web app using Argo CD

  - Create simple NGINX web app with appropriate manifests
  - Create appropriate GitHub repo with **at least one commit**
https://github.com/alkon/asg-8-argocd-repo.git
```
asg-8-argocd-repo/
└── manifests/
    ├── deployment.yaml
    └── service.yaml
```
`c. ` Create the Argo CD `nginx-app` application using CLI

 ``` bash
    argocd app create nginx-app \
      --repo https://github.com/alkon/asg-8-argocd-repo.git \
      --path manifests \
      --dest-server https://kubernetes.default.svc \
      --dest-namespace default \
      --sync-policy automated \
      --self-heal \
      --auto-prune
```
   - Output: `application 'nginx-app' created`
   - Result: appears in Argo CD Dashboard
   - Note: alternatively the app may be created using Argo CD UI
   
---
#### Task 3: Deploy an OCI-based Helm Chart using Argo CD
 **Note**: Used the Bitnami's `external-dns` Helm chart (ver. 8.8.2)

`a. ` Register OCI Helm Registry with Argo CD
   - Add `secrets/bitnami-oci-repo.yaml`
  ```yaml
  stringData:
    url: registry-1.docker.io/bitnamicharts
    name: bitnamicharts
    type: helm
    enableOCI: "true"
```
   - Apply the Secret
```bash
   kubectl apply -f secrets/bitnami-oci-repo.yaml
```
- Note: This secret does 2 things:
  - Sets registry-1.docker.io/bitnamicharts as a Helm OCI repo
  - Enables OCI support with enableOCI: "true"

`b. `  Add Argo CD Application manifest `manifests/external-dns-app.yaml`:
```yaml
spec:
  destination:
    namespace: cert-manager # Target namespace in the cluster 
    server: https://kubernetes.default.svc # Points to Kubernetes API
  source:
    repoURL: registry-1.docker.io/bitnamicharts # OCI registry 
    chart: external-dns # Helm chart name
    targetRevision: 8.8.2
  project: default
  syncPolicy:
    automated:
      prune: true # Delete resources not defined in Git
      selfHeal: true # Automatically fix drifted resources
    syncOptions:
      - CreateNamespace=true 
```
- Note 1:  The `cert-manager` namespace will be created automatically (due to CreateNamespace=true)
- Note 2: `project: default` associates the app with the Argo CD `default` project (RBAC, resource limits, etc.)
- Note 3: `spec.syncPolicy` **fully-automated** sync GitOps flow with **pruning** and **healing**
- Note 4: When `selfHeal: true` is enabled in the sync policy, Argo CD:
  - Continuously compares the cluster live state with the desired state Helm/OCI source or Git.
  - If any resource is manually modified/deleted in the cluster, Argo CD automatically reverts it to match the source of truth.

- Apply the Application manifest
```bash
   kubectl apply -f manifests/external-dns-app.yaml
```

`c. ` Test Manual and Auto-Sync
- Before testing `auto-sync` we'll check if manual sync works
```bash
   argocd app sync external-dns-cf 
```
- Confirm that all expected resources are created and healthy
```bash
   kubectl get all -n cert-manager
```
- Identify Argo CD resource that may be **restored** after deletion 
```bash
   argocd app get external-dns-cf # List Argo CD managed resources
```
  - Note: Choose resource to be re-created quickly, i.e. will not crash the external-dns Pod
  It may be Service, ServiceAccount, NetworkPolicy, ClusterRole and few others.
```bash
   kubectl delete serviceaccount external-dns-cf -n cert-manager
```
  - Output: ServiceAccount restored after few seconds. 
  - It may be seen when running again `argocd app get external-dns-cf` or in Argo CD UI
  - Note: the next resources to be restored (if deleted)/healed (if modified)/pruned  - if their **Status**:
    - `Synced` - restored/healed/**not** pruned
    - `OutOfSync` - restored/healed/**not** pruned
    - `Missing` - restored/**not** healed/**not** pruned
    - `Orphaned` - **not** restored/**not** healed/pruned
    if defined:
    ```yaml
    syncPolicy:
    automated:
      prune: true # Delete resources not defined in Git
      selfHeal: true # Automatically fix drifted resources
    syncOptions:
      - CreateNamespace=true 
    ```
- Note: to process `update/healing` (not full resource re-creation) change replicaSet of the Deployment
  - First, found the real number replicas running (READY column):
  - Second, **auto-correct** the replicas number back to this from the source (on the OCI Registry Repo) number - here 1
``` bash
  kubectl get deployment external-dns-cf -n cert-manager
  kubectl scale deployment external-dns-cf -n cert-manager --replicas=3
```
---
#### Task 4: Use Sync Waves and Hooks

a. Create a multi-resource Deployment
   - Add dependencies in the app (e.g., ConfigMap used by a Deployment) 

   - Used Bitnami's OCI-based Helm chart, so added a `source.helm.values` and `source.helm.extraDeploy` blockes in the Argo CD Application manifest:
 ```yaml
source:
  helm:
    values: |
      extraEnvVars:
        - name: MY_CONFIG
          valueFrom:
            configMapKeyRef:
              name: external-dns-extra-config
              key: setting
  
    extraDeploy:
      - apiVersion: v1
        kind: ConfigMap
        metadata:
          name: external-dns-extra-config
        data:
          setting: "enabled"
```
 - `extraEnvVars` and `extraDeploy` are special Bitnami’s patterns
   - `extraEnvVars` injects an env var into the Deployment
   - `extraDeploy` creates a supporting ConfigMap
 - Apply Changes in the Application manifest:
```bash
   kubectl apply -f manifests/external-dns-app.yaml
```
 - Test After Applying via CLI or Argo CD UI:
```bash
   kubectl get configmap external-dns-extra-config -n cert-manager
```
 - **Summary**: What's modified - the rendered output that Helm generates before Argo CD applies it to the cluster.

b. Use **sync waves** ro ensure the ConfigMap  is applied before the Deployment
 - Note 1: in `extraDeploy.metadata.annotations` added `sync-wave` value of `-1`
 - Note 2: the sync-wave of Deployment has its defalt value of **0**, so the ConfigMap will be applied first 
```yaml
   extraDeploy:
     - apiVersion: v1
       kind: ConfigMap
       metadata:
         name: external-dns-extra-config
         annotations:
           argocd.argoproj.io/sync-wave: "-1"
       data:
         setting: "enabled"
```
 - Note 3: Not all Helm chart providers include helpful structures within their values.yaml (like Bitnami's source.helm.values.extraDeploy) 
that allow to directly inject Kubernetes manifests into the deployment process.
In this case `Kustomize` comes to help: it allows to take an existing Kubernetes manifest (including those generated by Helm) and apply overlays on top of it.

c. Implement a PreSync hook to perform a custom action before syncing the app
  - Note 1: Using the same Bitnami's `external-dns` Helm chart provided hybrid approach:
from one hand we continue to use the OCI registry source, but for managing hook's job the GitHub repo used:
`https://github.com/alkon/external-dns-gitops`
     
  - Note 2: to achieve this behaviour used `spec.sources` block, so
```yaml
   sources:   
     - repoURL: registry-1.docker.io/bitnamicharts
       chart: external-dns
       targetRevision: 8.8.2
       #helm.values - optional
     - repoURL: https://github.com/your-org/external-dns-gitops.git
       path: hook-job
       targetRevision: HEAD   
```
  - Note 3: the order of ops:
   a) Argo CD runs PreSync Job from the hook-job folder. b) Then it will pull and apply the external-dns chart from the OCI registry
   c) Namespace cert-manager will be created automatically thanks to CreateNamespace=true.

  
  
  