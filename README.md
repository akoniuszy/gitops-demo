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
  # Dedykowany ServiceAccount do deploymentu (impersonacja)
  destinationServiceAccounts:
    - server: https://kubernetes.default.svc
      namespace: dlh-lab
      defaultServiceAccount: argocd-deployer-dlh-lab
EOF
```

Zabezpieczenia AppProject:

| Zagrożenie | Ochrona |
|---|---|
| Deploy do innego namespace | `destinations` — tylko `dlh-lab` |
| Tworzenie zasobów klastrowych | `clusterResourceWhitelist: []` — puste |
| Eskalacja uprawnień (RoleBinding) | `namespaceResourceBlacklist: rbac.authorization.k8s.io/*` |
| Użycie obcego repo Git | `sourceRepos` — tylko `https://github.com/akoniuszy/*` |
| Szerokie uprawnienia kontrolera | `destinationServiceAccounts` — impersonacja dedykowanego SA |

### 3. Dedykowany ServiceAccount do deploymentu

Zamiast dawać kontrolerowi ArgoCD szeroki `edit` do namespace, tworzymy dedykowany SA z minimalnymi uprawnieniami. Kontroler ArgoCD **impersonuje** tego SA przy syncu.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-deployer-dlh-lab
  namespace: dlh-lab
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-deployer
  namespace: dlh-lab
rules:
  - apiGroups: [""]
    resources: ["services", "configmaps", "secrets", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["route.openshift.io"]
    resources: ["routes"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods", "pods/log", "events"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-deployer-dlh-lab
  namespace: dlh-lab
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argocd-deployer
subjects:
  - kind: ServiceAccount
    name: argocd-deployer-dlh-lab
    namespace: dlh-lab
EOF
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
2. Utwórz dedykowany AppProject (jak w kroku 2) z `destinations`, `sourceNamespaces` i `destinationServiceAccounts` ustawionym na nowy namespace
3. Utwórz dedykowany ServiceAccount + Role + RoleBinding w nowym namespace (jak w kroku 3)

**Nie dodawaj** nowych namespace'ów do projektu `default`.
**Nie używaj** `--clusterrole=edit` — zawsze twórz dedykowany SA z minimalnymi uprawnieniami.

---

## Instrukcja: ArgoCD dla nowego namespace

Poniżej kompletna instrukcja krok po kroku. Zamień `<NAMESPACE>` na nazwę namespace i `<REPO_PATTERN>` na wzorzec repozytoriów Git (np. `https://github.com/my-org/*`).

### Krok 1 — Dodaj namespace do ArgoCD CR

```bash
# Pobierz aktualną listę
CURRENT=$(oc get argocd openshift-gitops -n openshift-gitops -o jsonpath='{.spec.sourceNamespaces}')
echo "Aktualne sourceNamespaces: $CURRENT"

# Dodaj nowy namespace (merge doda do istniejącej listy)
oc patch argocd openshift-gitops -n openshift-gitops --type json \
  -p '[{"op":"add","path":"/spec/sourceNamespaces/-","value":"<NAMESPACE>"}]'
```

### Krok 2 — Utwórz AppProject

```bash
cat <<'EOF' | sed 's/<NAMESPACE>/NAZWA_NAMESPACE/g; s/<REPO_PATTERN>/WZORZEC_REPO/g' | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: <NAMESPACE>
  namespace: openshift-gitops
spec:
  description: "Projekt ArgoCD dla namespace <NAMESPACE>"
  destinations:
    - namespace: <NAMESPACE>
      server: https://kubernetes.default.svc
  sourceRepos:
    - '<REPO_PATTERN>'
  sourceNamespaces:
    - <NAMESPACE>
  clusterResourceWhitelist: []
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
  namespaceResourceBlacklist:
    - group: rbac.authorization.k8s.io
      kind: '*'
  destinationServiceAccounts:
    - server: https://kubernetes.default.svc
      namespace: <NAMESPACE>
      defaultServiceAccount: argocd-deployer-<NAMESPACE>
EOF
```

### Krok 3 — Utwórz ServiceAccount, Role i RoleBinding

```bash
cat <<'EOF' | sed 's/<NAMESPACE>/NAZWA_NAMESPACE/g' | oc apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-deployer-<NAMESPACE>
  namespace: <NAMESPACE>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-deployer
  namespace: <NAMESPACE>
rules:
  - apiGroups: [""]
    resources: ["services", "configmaps", "secrets", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["route.openshift.io"]
    resources: ["routes"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods", "pods/log", "events"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-deployer-<NAMESPACE>
  namespace: <NAMESPACE>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argocd-deployer
subjects:
  - kind: ServiceAccount
    name: argocd-deployer-<NAMESPACE>
    namespace: <NAMESPACE>
EOF
```

### Krok 4 — Utwórz Application

Developerzy mogą teraz tworzyć Applications w swoim namespace:

```bash
cat <<'EOF' | sed 's/<NAMESPACE>/NAZWA_NAMESPACE/g' | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: moja-aplikacja
  namespace: <NAMESPACE>
spec:
  project: <NAMESPACE>
  source:
    repoURL: https://github.com/my-org/my-app
    targetRevision: main
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: <NAMESPACE>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

### Weryfikacja

```bash
# Sprawdź AppProject
oc get appproject <NAMESPACE> -n openshift-gitops -o jsonpath='
Destinations:  {.spec.destinations}
SourceRepos:   {.spec.sourceRepos}
ServiceAccounts: {.spec.destinationServiceAccounts}
'

# Sprawdź SA i uprawnienia
oc get sa,role,rolebinding -n <NAMESPACE> | grep argocd-deployer

# Sprawdź Application
oc get application -n <NAMESPACE>
```

### Checklist

- [ ] Namespace istnieje (`oc new-project <NAMESPACE>` lub `oc create namespace <NAMESPACE>`)
- [ ] ArgoCD CR ma namespace w `sourceNamespaces`
- [ ] AppProject utworzony w `openshift-gitops` z ograniczeniami
- [ ] SA `argocd-deployer-<NAMESPACE>` utworzony w namespace
- [ ] Role i RoleBinding utworzone w namespace
- [ ] Namespace NIE dodany do AppProject `default`
