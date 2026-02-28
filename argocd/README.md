## Chart `infra/argocd` – Wrapper Helm pour Argo CD

Ce chart est un **wrapper** autour du chart officiel [`argo/argo-cd`] et permet d'installer Argo CD facilement dans ton cluster Kubernetes (k3d/kind, etc.).  
Dans le workflow actuel, l'UI et l'API Argo CD sont accessibles via un **port‑forward** démarré par `run.sh` sur `http://localhost:9090`.

---

### 1. Pré‑requis

- Un cluster Kubernetes accessible (`kubectl get nodes` OK)
- `helm` installé
- (Optionnel) Un Ingress Controller si tu veux un domaine personnalisé plus tard  
  → **mais par défaut ce chart utilise uniquement un NodePort** (pas besoin de TLS/mkcert).

---

### 2. Namespace `argocd`

Depuis `infra` :

```bash
cd ~/Documents/projets/infra

kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
```

---

### 3. Installer / mettre à jour Argo CD

Depuis ce répertoire :

```bash
cd ~/Documents/projets/infra/argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm dependency update

helm upgrade --install argocd . \
  -n argocd \
  --create-namespace
```

Vérifier :

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd argocd-server
```

Le service `argocd-server` peut être de type **NodePort** en interne, mais l'accès depuis ta machine se fait via port‑forward (voir ci‑dessous).

---

### 4. Accéder à l’UI Argo CD

Le script `run.sh` démarre automatiquement un port‑forward :

```bash
./run.sh
```

Puis ouvre simplement dans ton navigateur :

```text
http://localhost:9090/applications
```

Identifiants par défaut :

- utilisateur : `admin`
- mot de passe initial :

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Ensuite, change le mot de passe dans l’UI Argo CD.

> 💡 **Intégration avec Jenkins**  
> L'intégration directe Jenkins → Argo CD nécessite une exposition réseau accessible depuis le conteneur Jenkins.  
> Pour l’instant, garde `ARGOCD_ENABLED="false"` dans le `Jenkinsfile` et utilise Argo CD en mode GitOps / via le CLI (`argocd ... --server localhost:9090`) pendant que `run.sh` tourne.
