# ArgoCD GitOps on VKS (adapted from papivot/argocd-gitops-tanzu)

Same GitOps idea as [papivot/argocd-gitops-tanzu](https://github.com/papivot/argocd-gitops-tanzu) —
ArgoCD drives creation of a workload Kubernetes cluster plus cert-manager/Contour add-ons — rewritten
for a VCF 9 / vSphere Kubernetes Service (VKS) Supervisor, where:

- ArgoCD is provisioned via the native `ArgoCD` Supervisor Service CR, not a manual `install.yaml`.
- The workload cluster uses the `builtin-generic-v3.x` ClusterClass, not the older `tanzukubernetescluster`
  TKGS ClusterClass the original repo targets (that ClusterClass doesn't exist on VKS Supervisors).

Tested against: vCenter 8.0.3 (VCF 9), Supervisor cluster `wld03-cls01`, `builtin-generic-v3.5.0`,
Kubernetes `v1.34.2+vmware.2`.

## Prerequisites

- A vSphere Namespace (`demo1` below) with VM classes `best-effort-medium`/`best-effort-small` and a
  storage policy assigned (`vsan-default-storage-policy` in this example).
- `kubectl` + the `kubectl-vsphere` plugin, logged into the Supervisor:
  ```bash
  kubectl vsphere login --insecure-skip-tls-verify --server <SUPERVISOR_API_SERVER> -u administrator@vsphere.local
  ```

## 1. Install ArgoCD (native Supervisor Service)

```bash
kubectl apply -f argocd/argocd-instance.yaml
kubectl get argocd -n demo1 -w      # wait for Running
kubectl get svc -n demo1 | grep argocd-server   # LoadBalancer IP
```

## 2. Log in to ArgoCD

```bash
argocd admin initial-password -n demo1
argocd login <ARGOCD_SERVER_LB_IP>
argocd account update-password
```

## 3. Deploy the workload cluster via GitOps

```bash
argocd app create tkc-deploy \
  --repo <YOUR_NEW_REPO_URL> \
  --path tkc \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace demo1 \
  --auto-prune --sync-policy auto

kubectl get cluster gitops-tkc1 -n demo1 -w   # wait for PHASE: Provisioned
```

## 4. Register the workload cluster and roll out add-ons

```bash
kubectl vsphere login --insecure-skip-tls-verify \
  --server <SUPERVISOR_API_SERVER> -u administrator@vsphere.local \
  --tanzu-kubernetes-cluster-namespace demo1 \
  --tanzu-kubernetes-cluster-name gitops-tkc1

argocd cluster add <gitops-tkc1-kubectl-context> --name gitops-tkc1
argocd cluster list   # note the "SERVER" value for gitops-tkc1

# fill in <WORKLOAD_CLUSTER_SERVER> in addons/*.yaml with that value, then:
kubectl apply -f addons/cert-manager-app.yaml
kubectl apply -f addons/contour-app.yaml
```

## Notes

- `cert-manager/` and `contour/` reference the upstream install manifests remotely via kustomize
  rather than vendoring them, to keep this repo small. If Docker Hub rate-limits any image pull,
  point the `images:` override in `contour/kustomization.yaml` at the Harbor Supervisor Service
  running in this environment (`svc-harbor-domain-c8`), the same workaround papivot's repo uses
  with its own Harbor instance.
- The original repo's `Dockerfile-synchook` PostSync-hook automation (auto-registering the new
  cluster into ArgoCD) is intentionally not replicated here for this first pass — the cluster is
  registered manually with `argocd cluster add` in step 4. Worth automating once this flow is proven.
