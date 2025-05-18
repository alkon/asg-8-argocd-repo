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
  - Note 4: To watch the hook before it deleted possible to change hook-delete-policy to:
`argocd.argoproj.io/hook-delete-policy: BeforeHookCreation`
```yaml
metadata:
  name: prepare-dns
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation #HookSucceeded
```
  - After re-apply and re-sync on pods list appear the hook's pod:
```bash
   kubectl get pods -n cert-manager
```
  - Output: `prepare-dns-7d9p5 0/1     Completed   0          26s` 
---
#### Task 5: Multi-cluster management (Hub-Spoke model)
  - Note: `k3d` clusters used here

 `a. ` Create cluster network named `k3d`
```bash
   docker network create k3d-net
```
 - Note: using a **named** network like `k3d-net` is strongly recommended because:

   - **Stable DNS resolution:**
   k3d injects CoreDNS NodeHosts entries that map each cluster node’s DNS name (e.g. k3d-target-server-0) to its Docker‐assigned IP. On Docker Desktop restarts these ephemeral networks usually change, which can break the mgmt-cluster→target-cluster connection. A named network survives restarts, so the IPs stay fixed.

   - **Easier cluster discovery:**
   When clusters share one network, you can refer to the target API server by name (e.g. https://k3d-target-server-0:6443) without extra port-forwards or host overrides.

   - **Better cleanup and isolation:**
   You can tear both clusters down with one docker network rm k3d-net, and you won’t accidentally collide with other containers you’re running on the default bridge.
   
`b. ` Create 2 clusters on the `k3d-net` network:
```bash
   k3d cluster create mgmt --network k3d-net --wait --agents 0
k3d cluster create target --network k3d-net --wait
```
  - Ensure the created clusters exists:
  ```bash
     kubectl config get-contexts
  ``` 
  Output: 
   ```text
      CURRENT   NAME                  CLUSTER               AUTHINFO                 NAMESPACE
                k3d-mgmt              k3d-mgmt              admin@k3d-mgmt           
      *         k3d-target            k3d-target            admin@k3d-target         
   ``` 
   - Note: The star (*) is on k3d-target, meaning you’re currently “inside” the target cluster.

   - Switch to the management cluster:
   ```bash
      kubectl config use-context k3d-mgmt
   ```
   - Verify cross-cluster connectivity:
```bash
  # Retrieve the spoke API server IP
echo "Retrieving spoke API server IP..."
export TARGET_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' k3d-target-server-0)
echo "TARGET_IP=$TARGET_IP"

# Launch a temporary pod in the hub cluster, passing TARGET_IP
echo "Testing connectivity from hub to spoke..."
kubectl run curl-test \
  --context k3d-mgmt \
  --image=curlimages/curl:latest \
  --restart=Never --rm -i --tty \
  --env TARGET_IP="$TARGET_IP" \
  -- sh

# Inside the pod, run:
ping -c3 "$TARGET_IP"                      # ICMP reachability
curl -vk "https://$TARGET_IP:6443/version"   # TLS handshake + API (401 = expected)

# Exit to terminate the pod
exit
```
   - Install Argo CD into the hub (k3d-mgmt):
```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   helm install argocd argo/argo-cd \
     --namespace argocd --create-namespace \
     --version $(helm search repo argo/argo-cd --versions \
       | awk 'NR==2{print $2}')
   
   # Rollout the server deployment
   kubectl rollout status deploy/argocd-server -n argocd
```
   - Prepare the **spoke** for Argo CD registration
- 1. Switch to the spoke cluster (k3d-target)
```bash
  kubectl config use-context k3d-target
```
- 2. On the spoke cluster, create a ServiceAccount in kube-system and bind cluster-admin (for demo purposes).
```bash
   # Create ServiceAccount
kubectl apply -n kube-system -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-manager
EOF

# Bind cluster-admin role
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-manager-rolebinding
subjects:
- kind: ServiceAccount
  name: argocd-manager
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

- 3. Grab the `spoke's` secret name, extract the bearer token and CA
```bash      
  SECRET=$(kubectl -n kube-system get sa argocd-manager -o jsonpath='{.secrets[0].name}')
  TOKEN=$(kubectl -n kube-system get secret $SECRET -o jsonpath='{.data.token}' | base64 -d)
  CA=$(kubectl -n kube-system get secret $SECRET -o jsonpath='{.data.ca\.crt}')
```
- 4. Switch back to the hub cluster (k3d-mgmt)
```bash
  kubectl config use-context k3d-mgmt
```
- 5. Register the spoke with Argo CD (applied on the `hub`)
```bash
   cat <<EOF | kubectl apply -n argocd -f -
apiVersion: v1
kind: Secret
metadata:
  name: k3d-target-cluster
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: k3d-target
  server: https://k3d-target-server-0:6443
  config: |
    {
      "bearerToken": "${TOKEN}",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "${CA}"
      }
    }
EOF
```

- 6. Port-forward and login.
   - Note: run in separate terminal
```bash
   kubectl --context k3d-mgmt port-forward svc/argocd-server -n argocd 8080:443
```
- 7. Login to the Argo CD
```bash
   # Extract admin password
export ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

# Log in via CLI
argocd login localhost:8080 \
  --username admin \
  --password "$ARGO_PWD" \
  --insecure
```

- 8. List Argo CD apps and clusters
```bash
   # List apps & clusters
argocd app list
argocd cluster list
```
- 9. Deploy and sync the appropraite OCI registry chart
  ```yaml
     # secrets/hub-cluster-oci-repo.yaml
     apiVersion: v1
        kind: Secret
        metadata:
          name: hub-oci-repo
          namespace: argocd
          labels:
            argocd.argoproj.io/secret-type: repository
        stringData:
          url: registry-1.docker.io/alkon100
          name: alkon100
          type: helm
          enableOCI: "true"
  ```
  ```yaml
     # manifests/busybox-test-app.yaml
     apiVersion: argoproj.io/v1alpha1
        kind: Application
        metadata:
          name: busybox-test-cf
          namespace: argocd
        spec:
          destination:
            namespace: cert-manager
              server: https://k3d-target-server-0:6443
        # test deploy on the hub cluster
        #    server: https://kubernetes.default.svc
            source:
              repoURL: registry-1.docker.io/alkon100
              chart: busybox-test-chart
              targetRevision: v1.0.1
            project: default
            syncPolicy:
              automated:
              prune: true
              selfHeal: true
              syncOptions:
                - CreateNamespace=true 
  ```

```bash
   kubectl apply -f secrets/hub-cluster-oci-repo.yaml
   kubectl apply -f manifests/busybox-test-app.yaml
```
- 10. Verify the workload on the **spoke** cluster:
```bash
    kubectl --context k3d-target -n argocd get pods
```
       
  
  
  