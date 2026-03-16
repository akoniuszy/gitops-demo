# GitOps Demo — hello-gitops

Przykładowa aplikacja zarządzana przez ArgoCD na OpenShift (OCP 4.20).  
Deployment nginx serwujący prostą stronę HTML, wdrażany automatycznie przez ArgoCD z tego repozytorium.

## Struktura

```
manifests/
  deployment.yaml   # Nginx (2 repliki, obraz ubi9/nginx-124, S2I run)
  service.yaml      # ClusterIP na porcie 8080
  route.yaml        # OpenShift Route
```

## Wymagania

Centralny ArgoCD w namespace `openshift-gitops` obsługuje Applications tworzone w namespace projektu (`dlh-lab`).

### 1. sourceNamespaces w ArgoCD CR

Kontroler ArgoCD musi obserwować Application CR w namespace projektu:

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge \
  -p '{"spec":{"sourceNamespaces":["dlh-lab"]}}'
```

### 2. sourceNamespaces w AppProject

Projekt ArgoCD musi pozwalać na Applications spoza `openshift-gitops`:

```bash
oc patch appproject default -n openshift-gitops --type merge \
  -p '{"spec":{"sourceNamespaces":["dlh-lab"]}}'
```

### 3. RoleBinding dla kontrolera ArgoCD

Kontroler musi mieć uprawnienia do zarządzania zasobami (Deployment, Service, Route) w namespace projektu:

```bash
oc create rolebinding argocd-controller-dlh-lab \
  --clusterrole=edit \
  --serviceaccount=openshift-gitops:openshift-gitops-argocd-application-controller \
  -n dlh-lab
```

> Operator GitOps automatycznie tworzy dodatkowy RoleBinding `openshift-gitops_dlh-lab` po dodaniu `sourceNamespaces` w ArgoCD CR.

## Tworzenie Application

```bash
cat <<'EOF' | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-demo
  namespace: dlh-lab
spec:
  project: default
  source:
    repoURL: https://github.com/akoniuszy/gitops-demo
    targetRevision: main
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: dlh-lab
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

## Testowanie GitOps

Zmień `replicas` lub treść HTML w `deployment.yaml`, push do Git — ArgoCD automatycznie zsynchronizuje zmiany.

```bash
# Sprawdź status aplikacji
curl -s http://hello-gitops-dlh-lab.apps.ocp.olympus.internal

# Wymuś natychmiastowy sync
oc annotate application gitops-demo -n dlh-lab argocd.argoproj.io/refresh=hard --overwrite
```

## Dodawanie kolejnych namespace'ów

Aby dodać ArgoCD do nowego namespace, powtórz kroki 1–3 z sekcji Wymagania, podmieniając `dlh-lab` na nowy namespace.
