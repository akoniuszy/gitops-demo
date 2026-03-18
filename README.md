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

### 3. Włączenie impersonacji w ArgoCD

Domyślnie ArgoCD **nie impersonuje** SA — kontroler działa jako swoje własne konto.  
Aby aktywować impersonację (żeby `destinationServiceAccounts` z AppProject faktycznie działało), ustaw klucz `application.sync.impersonation.enabled` w `argocd-cm`.

**Ważne:** ConfigMap `argocd-cm` jest zarządzany przez operator GitOps — bezpośredni `oc patch cm` zostanie nadpisany.  
Należy użyć pola `extraConfig` w ArgoCD CR:

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge \
  -p '{"spec":{"extraConfig":{"application.sync.impersonation.enabled":"true"}}}'
```

Weryfikacja:
```bash
# Sprawdź czy klucz trafił do argocd-cm
oc get cm argocd-cm -n openshift-gitops -o jsonpath='{.data.application\.sync\.impersonation\.enabled}'
# Powinno zwrócić: true
```

> Ten krok wykonujesz **raz** — dotyczy całej instancji ArgoCD, nie per namespace.

### 4. Dedykowany ServiceAccount do deploymentu

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

### 5. Uprawnienia do impersonacji SA w namespace

Kontroler ArgoCD potrzebuje verbu `impersonate` na docelowym ServiceAccount.  
Kubernetes wymaga tego jawnie — samo `destinationServiceAccounts` w AppProject to tylko konfiguracja ArgoCD, ale k8s API musi pozwolić kontrolerowi działać "w imieniu" tego SA.

Tworzymy namespaced Role + RoleBinding:

```bash
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
```

> **Bezpieczeństwo:** `resourceNames` ogranicza impersonację **tylko** do konkretnego SA (`argocd-deployer-dlh-lab`).  
> Kontroler nie może impersonować żadnego innego konta w tym namespace.

### Weryfikacja impersonacji

Po wykonaniu kroków 3–5, sprawdź w audit logach API servera, że kontroler faktycznie impersonuje dedykowany SA:

```bash
# Szukaj w audit logu na dowolnym control plane
for node in $(oc get nodes --selector=node-role.kubernetes.io/master -o name | sed 's|node/||'); do
  echo "=== $node ==="
  oc adm node-logs $node --path=kube-apiserver/audit.log 2>&1 | \
    grep 'hello-gitops' | grep '"create"' | grep 'deployments' | tail -1 | \
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
impersonating: system:serviceaccount:dlh-lab:argocd-deployer-dlh-lab
```

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

1. Dodaj namespace do `sourceNamespaces` w ArgoCD CR (krok 1)
2. Utwórz dedykowany AppProject z `destinationServiceAccounts` (krok 2)
3. Włącz impersonację — jednorazowo, jeśli jeszcze nie włączona (krok 3)
4. Utwórz dedykowany ServiceAccount + Role + RoleBinding (krok 4)
5. Utwórz Role + RoleBinding na verb `impersonate` (krok 5)

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

### Krok 3 — Włącz impersonację (jednorazowo)

> **Jeśli już wykonałeś ten krok** dla innego namespace — pomiń go. Ustawienie jest globalne.

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type merge \
  -p '{"spec":{"extraConfig":{"application.sync.impersonation.enabled":"true"}}}'
```

### Krok 4 — Utwórz ServiceAccount, Role i RoleBinding

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

### Krok 5 — Uprawnienia do impersonacji SA

```bash
cat <<'EOF' | sed 's/<NAMESPACE>/NAZWA_NAMESPACE/g' | oc apply -f -
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

> `resourceNames` ogranicza impersonację wyłącznie do SA `argocd-deployer-<NAMESPACE>` w danym namespace.

### Krok 6 — Utwórz Application

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
- [ ] AppProject utworzony w `openshift-gitops` z ograniczeniami i `destinationServiceAccounts`
- [ ] Impersonacja włączona w ArgoCD CR (`extraConfig: application.sync.impersonation.enabled: "true"`) — jednorazowo
- [ ] SA `argocd-deployer-<NAMESPACE>` utworzony w namespace
- [ ] Role `argocd-deployer` i RoleBinding utworzone w namespace
- [ ] Role `allow-argocd-impersonate-deployer` + RoleBinding na verb `impersonate` utworzone w namespace
- [ ] Namespace NIE dodany do AppProject `default`
