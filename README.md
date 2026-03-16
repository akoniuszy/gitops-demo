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

## Konfiguracja ArgoCD

Centralny ArgoCD w namespace `openshift-gitops` obsługuje Applications tworzone w namespace projektu (`dlh-lab`).

### 1. sourceNamespaces w ArgoCD CR

Kontroler ArgoCD musi obserwować Application CR w namespace projektu:

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge \
  -p '{"spec":{"sourceNamespaces":["dlh-lab"]}}'
```

### 2. Dedykowany AppProject (zamiast default)

**Nie używaj projektu `default`** — ma pełne uprawnienia do wszystkich namespace'ów i zasobów klastrowych.  
Utwórz dedykowany AppProject z ograniczeniami:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: dlh-lab
  namespace: openshift-gitops
spec:
  description: "Projekt zespolu DLH - tylko namespace dlh-lab"
  # Tylko namespace dlh-lab na lokalnym klastrze
  destinations:
    - namespace: dlh-lab
      server: https://kubernetes.default.svc
  # Tylko repozytoria zespolu
  sourceRepos:
    - 'https://github.com/akoniuszy/*'
  # Applications z namespace dlh-lab
  sourceNamespaces:
    - dlh-lab
  # BRAK zasobow klastrowych (Namespace, ClusterRole, CRD itp.)
  clusterResourceWhitelist: []
  # Dozwolone typy zasobow w namespace
  namespaceResourceWhitelist:
    - group: ''
      kind: ConfigMap
    - group: ''
      kind: Service
    - group: ''
      kind: Secret
    - group: ''
      kind: PersistentVolumeClaim
    - group: apps
      kind: Deployment
    - group: apps
      kind: StatefulSet
    - group: apps
      kind: ReplicaSet
    - group: route.openshift.io
      kind: Route
    - group: networking.k8s.io
      kind: Ingress
    - group: networking.k8s.io
      kind: NetworkPolicy
    - group: batch
      kind: Job
    - group: batch
      kind: CronJob
    - group: autoscaling
      kind: HorizontalPodAutoscaler
  # Blokada zasobow RBAC (zapobiega eskalacji uprawnien)
  namespaceResourceBlacklist:
    - group: rbac.authorization.k8s.io
      kind: '*'
EOF
```

Zabezpieczenia AppProject:

| Zagrożenie | Ochrona |
|---|---|
| Deploy do innego namespace | `destinations` — tylko `dlh-lab` |
| Tworzenie zasobów klastrowych | `clusterResourceWhitelist: []` — puste |
| Eskalacja uprawnień (RoleBinding) | `namespaceResourceBlacklist: rbac.authorization.k8s.io/*` |
| Użycie obcego repo Git | `sourceRepos` — tylko `https://github.com/akoniuszy/*` |

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

Application **musi** wskazywać na projekt `dlh-lab` (nie `default`):

```bash
cat <<'EOF' | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-demo
  namespace: dlh-lab
spec:
  project: dlh-lab
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

Aby dodać ArgoCD do nowego namespace (np. `team-xyz`):

1. Dodaj namespace do `sourceNamespaces` w ArgoCD CR
2. Utwórz dedykowany AppProject (jak w kroku 2) z `destinations` i `sourceNamespaces` ustawionym na nowy namespace
3. Utwórz RoleBinding dla kontrolera ArgoCD w nowym namespace (jak w kroku 3)

**Nie dodawaj** nowych namespace'ów do projektu `default`.
