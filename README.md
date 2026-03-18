# ArgoCD Multi-Tenant Setup na OpenShift

Procedura onboardingu namespace do centralnego ArgoCD (OpenShift GitOps) z impersonacją dedykowanego ServiceAccount.

**Środowisko:** OCP 4.20 + OpenShift GitOps 1.19 (ArgoCD v3.x)

---

## 1. Cel dokumentu

Zapewnić **bezpieczny, powtarzalny** proces dodawania nowych namespace'ów do centralnego ArgoCD, gdzie:

- Każdy zespół ma **własny namespace** i **własny AppProject**
- Kontroler ArgoCD **nie działa** ze swoimi szerokimi uprawnieniami — **impersonuje** dedykowany SA z minimalnymi prawami
- Zespół **nie może** deployować poza swój namespace, tworzyć zasobów klastrowych ani modyfikować RBAC
- Developerzy tworzą Applications **w swoim namespace** (nie w `openshift-gitops`)

---

## 2. Model docelowy

```
┌──────────────────────────────────────────────────────────────────┐
│  openshift-gitops (centralny)                                    │
│                                                                  │
│  ArgoCD CR                                                       │
│    sourceNamespaces: [dlh-lab, team-xyz, ...]                    │
│    extraConfig:                                                  │
│      application.sync.impersonation.enabled: "true"              │
│                                                                  │
│  AppProject: dlh-lab             AppProject: team-xyz            │
│    destinations: [dlh-lab]         destinations: [team-xyz]      │
│    destinationServiceAccounts:     destinationServiceAccounts:   │
│      → argocd-deployer-dlh-lab       → argocd-deployer-team-xyz │
└──────────────────────┬───────────────────────┬───────────────────┘
                       │ impersonuje           │ impersonuje
                       ▼                       ▼
┌──────────────────────────────┐ ┌─────────────────────────────────┐
│  namespace: dlh-lab          │ │  namespace: team-xyz            │
│                              │ │                                 │
│  SA: argocd-deployer-dlh-lab │ │  SA: argocd-deployer-team-xyz  │
│  Role: argocd-deployer       │ │  Role: argocd-deployer         │
│  Role: allow-...-impersonate │ │  Role: allow-...-impersonate   │
│                              │ │                                 │
│  Application: gitops-demo    │ │  Application: my-app           │
│  Deployment, Service, Route  │ │  Deployment, Service, Route    │
└──────────────────────────────┘ └─────────────────────────────────┘
```

**Przepływ syncu:**
1. ArgoCD controller wykrywa drift / nowy commit
2. Controller **impersonuje** SA `argocd-deployer-<NS>` (verb `impersonate` w k8s RBAC)
3. API server sprawdza uprawnienia **impersonowanego SA** (Role `argocd-deployer`)
4. Zasoby tworzone/aktualizowane z minimalnymi uprawnieniami

**Zabezpieczenia AppProject:**

| Zagrożenie | Ochrona |
|---|---|
| Deploy do innego namespace | `destinations` — tylko docelowy NS |
| Tworzenie zasobów klastrowych | `clusterResourceWhitelist: []` |
| Eskalacja uprawnień (RoleBinding) | `namespaceResourceBlacklist: rbac.authorization.k8s.io/*` |
| Użycie obcego repo Git | `sourceRepos` — ograniczone do wzorca |
| Szerokie uprawnienia kontrolera | `destinationServiceAccounts` + impersonacja |

---

## 3. Parametry do podmiany

W procedurze poniżej zamień:

| Placeholder | Opis | Przykład |
|---|---|---|
| `<NAMESPACE>` | Nazwa namespace zespołu | `dlh-lab` |
| `<REPO_PATTERN>` | Wzorzec repozytoriów Git | `https://github.com/akoniuszy/*` |

---

## 4. Procedura krok po kroku

### Krok 1 — Utwórz namespace

```bash
oc new-project <NAMESPACE>
# lub:
oc create namespace <NAMESPACE>
```

### Krok 2 — Dodaj namespace do ArgoCD CR

**Pierwszy namespace** (lista `sourceNamespaces` jeszcze nie istnieje) — merge patch:

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge \
  -p '{"spec":{"sourceNamespaces":["<NAMESPACE>"]}}'
```

**Kolejny namespace** (lista już istnieje) — JSON Patch dopisuje do istniejącej tablicy:

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type json \
  -p '[{"op":"add","path":"/spec/sourceNamespaces/-","value":"<NAMESPACE>"}]'
```

> Operator GitOps automatycznie tworzy RoleBinding `openshift-gitops_<NAMESPACE>` w nowym namespace (uprawnienia do zarządzania Application CR).

### Krok 3 — Włącz impersonację (jednorazowo)

> **Pomiń**, jeśli już włączone dla innego namespace. Ustawienie jest globalne.

Domyślnie kontroler ArgoCD działa jako swoje własne konto. Aby `destinationServiceAccounts` działało, impersonacja musi być włączona.

**Ważne:** ConfigMap `argocd-cm` jest zarządzany przez operator — bezpośredni `oc patch cm` zostanie nadpisany.
Użyj pola `extraConfig` w ArgoCD CR:

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge \
  -p '{"spec":{"extraConfig":{"application.sync.impersonation.enabled":"true"}}}'
```

### Krok 4 — Utwórz AppProject

```bash
cat <<'EOF' | oc apply -f -
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

### Krok 5 — Utwórz ServiceAccount, Role i RoleBinding

```bash
cat <<'EOF' | oc apply -f -
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

### Krok 6 — Uprawnienia do impersonacji SA

Kontroler ArgoCD potrzebuje verbu `impersonate` na docelowym SA.
Samo `destinationServiceAccounts` w AppProject to konfiguracja ArgoCD — k8s API musi **osobno** pozwolić kontrolerowi działać w imieniu tego SA.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: allow-argocd-impersonate-deployer
  namespace: <NAMESPACE>
rules:
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["impersonate"]
    resourceNames: ["argocd-deployer-<NAMESPACE>"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: allow-argocd-controller-impersonate
  namespace: <NAMESPACE>
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: allow-argocd-impersonate-deployer
EOF
```

> **Bezpieczeństwo:** `resourceNames` ogranicza impersonację **wyłącznie** do SA `argocd-deployer-<NAMESPACE>`.
> Kontroler nie może impersonować żadnego innego konta w tym namespace.

### Krok 7 — Utwórz Application

Developerzy mogą teraz tworzyć Applications w swoim namespace:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: moja-aplikacja
  namespace: <NAMESPACE>
spec:
  project: <NAMESPACE>
  source:
    repoURL: <REPO_URL>
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

---

## 5. Weryfikacja

### Sprawdź konfigurację

```bash
# Impersonacja włączona?
oc get cm argocd-cm -n openshift-gitops \
  -o jsonpath='{.data.application\.sync\.impersonation\.enabled}'
# → true

# AppProject
oc get appproject <NAMESPACE> -n openshift-gitops -o jsonpath='
  destinations:       {.spec.destinations}
  sourceRepos:        {.spec.sourceRepos}
  serviceAccounts:    {.spec.destinationServiceAccounts}
'

# SA i RBAC w namespace
oc get sa,role,rolebinding -n <NAMESPACE> | grep -E 'argocd-deployer|impersonate'

# Application
oc get application -n <NAMESPACE>
```

### Sprawdź impersonację w audit logach

```bash
for node in $(oc get nodes --selector=node-role.kubernetes.io/master \
    -o name | sed 's|node/||'); do
  echo "=== $node ==="
  oc adm node-logs $node --path=kube-apiserver/audit.log 2>&1 | \
    grep '"create"' | grep 'deployments' | grep '<NAMESPACE>' | tail -1 | \
    python3 -c "
import sys,json
for line in sys.stdin:
    e = json.loads(line.strip())
    u = e.get('user',{}).get('username','?')
    imp = e.get('impersonatedUser',{}).get('username','')
    print(f'  user: {u}')
    print(f'  impersonating: {imp}' if imp else '  NO impersonation')
"
done
```

Oczekiwany wynik:
```
user: system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
impersonating: system:serviceaccount:<NAMESPACE>:argocd-deployer-<NAMESPACE>
```

### Test bezpieczeństwa — namespace escape

Zmień `namespace` w manifeście na inny namespace, push do Git. ArgoCD powinno odrzucić sync:

```
namespace <INNY_NS> is not permitted in project '<NAMESPACE>'
```

---

## 6. Checklist

- [ ] Namespace istnieje
- [ ] ArgoCD CR: namespace w `sourceNamespaces`
- [ ] ArgoCD CR: `extraConfig.application.sync.impersonation.enabled: "true"` (jednorazowo)
- [ ] AppProject w `openshift-gitops`: `destinations`, `sourceRepos`, `sourceNamespaces`, `destinationServiceAccounts`
- [ ] SA `argocd-deployer-<NAMESPACE>` w namespace
- [ ] Role `argocd-deployer` + RoleBinding w namespace
- [ ] Role `allow-argocd-impersonate-deployer` + RoleBinding (verb `impersonate`) w namespace
- [ ] Application w namespace zespołu (nie w `openshift-gitops`)
- [ ] Namespace **NIE** dodany do AppProject `default`

---

## 7. Przykład: demo `dlh-lab`

Konkretna realizacja procedury dla namespace `dlh-lab` z repozytorium `https://github.com/akoniuszy/gitops-demo`.

### Struktura manifestów

```
manifests/
  deployment.yaml   # Nginx (2 repliki, ubi9/nginx-124, S2I run)
  service.yaml      # ClusterIP na porcie 8080
  route.yaml        # OpenShift Route → hello-gitops-dlh-lab.apps.ocp.olympus.internal
```

### Zastosowane parametry

| Placeholder | Wartość |
|---|---|
| `<NAMESPACE>` | `dlh-lab` |
| `<REPO_PATTERN>` | `https://github.com/akoniuszy/*` |

### Komendy (wykonane wg procedury)

```bash
# Krok 1 — Namespace
oc new-project dlh-lab

# Krok 2 — sourceNamespaces
oc patch argocd openshift-gitops -n openshift-gitops --type merge \
  -p '{"spec":{"sourceNamespaces":["dlh-lab"]}}'

# Krok 3 — Impersonacja (jednorazowo)
oc patch argocd openshift-gitops -n openshift-gitops --type merge \
  -p '{"spec":{"extraConfig":{"application.sync.impersonation.enabled":"true"}}}'

# Krok 4 — AppProject
cat <<'EOF' | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: dlh-lab
  namespace: openshift-gitops
spec:
  description: "Projekt zespolu DLH - tylko namespace dlh-lab"
  destinations:
    - namespace: dlh-lab
      server: https://kubernetes.default.svc
  sourceRepos:
    - 'https://github.com/akoniuszy/*'
  sourceNamespaces:
    - dlh-lab
  clusterResourceWhitelist: []
  namespaceResourceBlacklist:
    - group: rbac.authorization.k8s.io
      kind: '*'
  destinationServiceAccounts:
    - server: https://kubernetes.default.svc
      namespace: dlh-lab
      defaultServiceAccount: argocd-deployer-dlh-lab
EOF

# Krok 5 — SA + Role + RoleBinding
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

# Krok 6 — Impersonate RBAC
cat <<'EOF' | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: allow-argocd-impersonate-deployer
  namespace: dlh-lab
rules:
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["impersonate"]
    resourceNames: ["argocd-deployer-dlh-lab"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: allow-argocd-controller-impersonate
  namespace: dlh-lab
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: allow-argocd-impersonate-deployer
EOF

# Krok 7 — Application
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

### Testowanie

```bash
# Status aplikacji
oc get application gitops-demo -n dlh-lab

# Strona
curl -s http://hello-gitops-dlh-lab.apps.ocp.olympus.internal

# Wymuś sync
oc annotate application gitops-demo -n dlh-lab \
  argocd.argoproj.io/refresh=hard --overwrite

# Test self-heal — usuń deployment, ArgoCD go odtworzy
oc delete deployment hello-gitops -n dlh-lab
# → po ~15s deployment wraca automatycznie
```
