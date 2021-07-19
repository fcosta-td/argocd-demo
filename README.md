# ArgoCD
Instuctions for implementating ArgoCD with MiniKube
https://argoproj.github.io/argo-cd/getting_started/

## Requirements
- minikube
- kubectl command-line tool.
- Have a kubeconfig file (default location is ~/.kube/config).
- helm with [helm-secrets](https://www.google.com) plugin
- [kustomize](https://github.com/kubernetes-sigs/kustomize)
- [gpg key](https://github.com/fcosta-td/argocd/tree/main/dockerfile/argocd_demo_gpg.asc) installed

## Index
* [Install requirements](#install-requirements)
* [Install ArgoCD](#install-argocd)
* [Examples](##examples)
    * [Simple](###simple)
    * [Helm](###helm)
    * [Kustomize](###kustomize)
    * [kustime Helm](###kustomizehelm)
    * [App of Apps](###app of apps)

## Summary
This is a demonstration of a GitOps pipeline, with several examples:
* Simple - Service deployment
* Helm - Using Helm Charts with SOPS encrypted secrets
* Kustomize - Helm templates with kustomization.yaml for simpler deployments

## Install requirements
* [Install Minikube](https://minikube.sigs.k8s.io/docs/start/)
* [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [Install helm](https://helm.sh/docs/intro/install/)
* [Install kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)

## Install ArgoCD
```
minikube start
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/fcosta-td/argocd/main/CRD/install.yaml
```

### Install argo cli
Debian:
```
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```
MAC:
```
brew install argocd
```
Windows:
```
format c: /QX :)
```

### Access ArgoCD API
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
Configure Ingress: https://argoproj.github.io/argo-cd/operator-manual/ingress/

Configure Port-forward
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
### Login Using The CLI
For Argo 2.x, follow these steps:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Using the username admin and the password from above, login to Argo CD's IP or hostname:
```
argocd login localhost:8080 --username admin --insecure
```
Change password
```
argocd account update-password
```
You can access Argo CD using port forwarding: add --port-forward-namespace argocd flag to every CLI command or set ARGOCD_OPTS environment variable: export ARGOCD_OPTS='--port-forward-namespace argocd':

### Usefull commands
### Sync application
Manually
```
argocd app get <APP_NAME>
argocd app sync <APP_NAME>
```
Auto-sync
```
argocd app set <APP_NAME> --sync-policy automated --auto-prune --self-heal
```
App Manifest
```
spec:
  syncPolicy:
    automated:
        prune: true
        selfHeal: true
```
**References:** (https://argoproj.github.io/argo-cd/user-guide/auto_sync/)

# Examples
### Simple
Detects multiples services/deployments in the same directory.

You can deploy it with Kubectl
```
kubectl apply -f ~/Documents/git/argocd/applications/simple.yaml
```
or you can deploy via argocd cli
```
argocd app create simple --repo https://github.com/fcosta-td/argocd.git --path examples/simple --dest-server https://kubernetes.default.svc --dest-namespace default --self-heal --sync-policy automated --sync-option Prune=true --sync-option frequency=1m
```
### Helm
This example uses helm with helm-secrets plugin to deploy a chart with encrypted values using sops. Inside /examples/charts/trinodb/cluster_1 directory is a encryped secrets.yaml. Although a gpg key was used for encryption, it can be configured to work with a KMS key. It's also possible to store secrets directly to Vault.

```
# Set this only once
export SOPS_GPG_EXEC="gpg"
export SOPS_PGP_FP="013D26E064C0758596AE93A986A1183C1DFC20F4"

# Encrypt an entire file
sops -e secrets.yaml
sops -d secrets.yaml

# Encrypt base on regex
sops --encrypt --encrypted-regex '^(.*mysql.properties.*)$' secrets.yaml
```

You can deploy it with Kubectl
```
kubectl apply -f ~/Documents/git/argocd/applications/trinodb_cluster_1.yaml
```
or you can deploy via argocd cli
```
argocd app create trinodb-cluster-1 --repo https://github.com/fcosta-td/argocd.git --path examples/helm/trinodb/cluster_1 --dest-server https://kubernetes.default.svc --dest-namespace default --self-heal --sync-policy automated --sync-option Prune=true --helm-version 3 --helm-set-string valueFiles='secrets.yaml'
```
### Kustomize
Example using kustomize, with customizations per environment.

There is also a wrapper available and a plugin to use encrypted file with sops integrated with Kustomize

You can deploy it with Kubectl
```
kubectl apply -f ~/Documents/git/argocd/applications/kustomize_nginx.yaml
```
or you can deploy via argocd cli
```

```

### KustomizeHelm
This example uses a helm chart template combined with kustomize.
You can deploy it with kubectl
```
kubectl apply -f ~/Documents/git/argocd/applications/kustomizehelm.yaml
```

Create template from helm
```
cd kusthelm
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm fetch --untar grafana/grafana
cd grafana
helm template grafana grafana/grafana --output-dir base --namespace default --values values.yaml
```

# App of Apps
Using this approach, we only need to deploy just one app directly to our K8s cluster either with kubectl or with terraform, and every deployment file created under applications directory, will automatically be applied to our cluster.

```
kubectl apply -f /home/filipecosta/Documents/git/argocd/app_of_apps/app_of_apps.yaml
```
