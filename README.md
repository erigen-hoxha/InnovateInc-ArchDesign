# Argo CD

## Assumptions
### **Based on the setup:**

- Source code repo has the Dockerfile and Helm chart.
- Build & push Docker images via CI.
- Helm chart is versioned in the repo.
- Separate values.yaml per environment (Values.development.yaml).
- Kubernetes cluster is already set up and reachable.
- ArgoCD is installed in the cluster.

## Structure for GitOps

```bash
opsfleet/
├── helm/
│   └── inovation-inc/
│       ├── Chart.yaml
│       ├── templates/
│       └── Values.development.yaml
```

### **ArgoCD Application Manifest**

```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: inovation-inc
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/opsfleet/inovation.git
    targetRevision: main
    path: helm/inovation
    helm:
      valueFiles:
        - Values.development.yaml
      parameters:
        - name: image.tag
          value: "latest"
  destination:
    server: https://kubernetes.default.svc
    namespace: opsfleet
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Applying the ArgoCD App

Assume I am logged in:
```bash
argocd app create -f inovation-inc-dev.yaml
```

### **ArgoCD will then:**

- Monitor the repo.
- Detect changes (like updated image tag or value changes).
- Automatically apply them to the Kubernetes cluster.



## CI Step to Update Image Tag in Git:

```bash
- script: |
    sed -i "s/tag:.*/tag: $(Build.BuildId)/" helm/inovation-inc/Values.development.yaml
    git config user.name "erigen-hoxha"
    git config user.email "erigen_hoxha@hotmail.com"
    git add helm/inovation-inc/Values.development.yaml
    git commit -m "Update image tag to $(Build.BuildId)"
    git push
  displayName: "Update image tag in Git for GitOps"
```

### **ArgoCD will:**

- Detect the new tag in Git.
- Re-sync the deployment.
- No need to explicitly run helm upgrade in the pipeline anymore.
