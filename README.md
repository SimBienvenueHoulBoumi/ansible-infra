# Ansible – GitOps (ArgoCD)

Automatisation **déclarative** et **idempotente** : installation d’ArgoCD (chart au même niveau qu’`ansible`) et création des Application CRs. Une seule source de vérité pour les variables : **`group_vars/all.yml`**.

## Prérequis

- Ansible 2.14+
- Collection `community.kubernetes` : `ansible-galaxy collection install -r requirements.yml`
- `kubectl`, `helm` et accès au cluster (KUBECONFIG ou contexte par défaut)

## Variables (group_vars/all.yml uniquement)

| Variable | Description |
|----------|-------------|
| `kube_context` | Contexte kubectl (ex. `kind-kind`) |
| `argocd_namespace` | Namespace ArgoCD |
| `argocd_destination_server` | Serveur de destination des applications (défaut in-cluster) |
| `argocd_applications` | Liste des applications (name, namespace, repo_url, path, helm_values, sync_policy) |

Les chemins du chart ArgoCD sont définis dans le playbook (chart au même niveau que le répertoire `ansible`).

## Utilisation

```bash
cd infra/ansible
ansible-playbook playbooks/gitops.yml
```

- Uniquement ArgoCD : `--tags install`
- Uniquement les applications : `--tags applications`

## Rôles

- **argocd_install** : vérification cluster, namespace, `helm dependency update` + `helm upgrade --install`, affichage du mot de passe admin.
- **argocd_applications** : construction des définitions Application à partir de `argocd_applications`, puis `k8s` state=present.

## Jenkins

Un **Jenkinsfile** dans ce répertoire permet d’exécuter le playbook depuis un job Jenkins (agent avec `kubectl` et `KUBECONFIG` monté). Le job doit cloner un repo dont la racine contient **infra/ansible** et **infra/argocd**. Paramètre optionnel : **ANSIBLE_TAGS** (`all`, `install`, `applications`).

## Intégration

- Scripts shell (`main.sh argocd-install`, etc.) restent utilisables.
- Port-forward non géré par Ansible : `./main.sh argocd-port-forward 8084` sur l’hôte.
- Jenkins continue à faire login ArgoCD, set image, sync.
