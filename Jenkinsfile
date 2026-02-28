// ======================================================================
// Pipeline Ansible GitOps – installation ArgoCD + applications
// À lancer depuis un job Jenkins dont le workspace contient infra/ansible (et infra/argocd).
// ======================================================================
pipeline {
    agent { node { label 'jenkins-agent' } }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        skipDefaultCheckout(false)
    }

    parameters {
        string(
            name: 'INFRA_REPO_URL',
            defaultValue: '',
            description: '(Optionnel) Si le repo cloné n\'a pas infra/, URL du repo infra à cloner (ex. https://github.com/user/mon-infra.git). La racine du repo doit contenir ansible/ et argocd/.'
        )
        string(
            name: 'INFRA_BRANCH',
            defaultValue: 'main',
            description: 'Branche du repo infra (si INFRA_REPO_URL est renseigné).'
        )
        string(
            name: 'ANSIBLE_DIR',
            defaultValue: 'infra/ansible',
            description: 'Chemin vers le répertoire Ansible. Par défaut infra/ansible.'
        )
        choice(
            name: 'ANSIBLE_TAGS',
            choices: ['all', 'install', 'applications'],
            description: 'Tags Ansible : all = playbook complet, install = ArgoCD uniquement, applications = Application CRs uniquement'
        )
    }

    environment {
        // kubeconfig monté par docker-compose dans l'agent : ${HOME}/.kube -> /home/jenkins/.kube (agent tourne en root, HOME=/root)
        KUBECONFIG = "/home/jenkins/.kube/config"
    }

    stages {
        stage('📥 Checkout') {
            steps {
                checkout scm
                script {
                    def hasAnsible = fileExists("${params.ANSIBLE_DIR}/playbooks/gitops.yml")
                    if (!hasAnsible && params.INFRA_REPO_URL?.trim()) {
                        echo "infra/ansible absent : clonage du repo infra (${params.INFRA_REPO_URL})..."
                        sh """
                            rm -rf infra
                            git clone --depth 1 -b '${params.INFRA_BRANCH}' '${params.INFRA_REPO_URL}' infra
                            test -d infra/ansible && test -f infra/ansible/playbooks/gitops.yml || (echo 'Le repo doit avoir ansible/ et argocd/ à la racine.' && exit 1)
                        """
                    }
                    // Repo type "ansible-infra" = playbooks à la racine ; sinon infra/ansible
                    if (fileExists("${params.ANSIBLE_DIR}/playbooks/gitops.yml")) {
                        env.ANSIBLE_WORKDIR = params.ANSIBLE_DIR
                    } else if (fileExists("playbooks/gitops.yml")) {
                        env.ANSIBLE_WORKDIR = "."
                    } else {
                        error("Playbook introuvable : ni ${params.ANSIBLE_DIR}/playbooks/gitops.yml ni playbooks/gitops.yml à la racine.")
                    }
                    echo "Répertoire Ansible : ${env.ANSIBLE_WORKDIR}"
                    sh "ls -la ${env.ANSIBLE_WORKDIR}/"
                }
            }
        }

        stage('🔧 Ansible') {
            steps {
                dir(env.ANSIBLE_WORKDIR) {
                    sh """
                        set -e
                        export PATH="/usr/local/bin:/usr/bin:\$PATH"
                        if ! command -v ansible-playbook >/dev/null 2>&1; then
                            echo "Ansible introuvable. Reconstruire l'image jenkins-agent : cd infra && docker compose build jenkins-agent && docker compose up -d jenkins-agent"
                            exit 127
                        fi
                        # Lib kubernetes requise par kubernetes.core (au cas où l'image n'a pas été reconstruite)
                        if ! python3 -c "import kubernetes" 2>/dev/null; then
                            echo "Installation de la lib Python kubernetes pour l'utilisateur courant..."
                            python3 -m pip install --user --break-system-packages kubernetes 2>/dev/null || python3 -m pip install --user kubernetes
                        fi
                        # Helm requis pour helm dependency update et le module helm (reconstruire l'image pour l'avoir à demeure)
                        if ! command -v helm >/dev/null 2>&1; then
                            echo "Installation de Helm..."
                            curl -sSLf https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
                        fi
                        helm version
                        # Depuis le conteneur, 127.0.0.1 = le conteneur ; patcher le kubeconfig et le passer à Ansible explicitement
                        ORIG_KUBE="\${KUBECONFIG:-/home/jenkins/.kube/config}"
                        KUBE_EXTRA=""
                        if [ -f "\$ORIG_KUBE" ] && grep -q '127.0.0.1' "\$ORIG_KUBE" 2>/dev/null; then
                            PATCHED="/tmp/kubeconfig-docker.yaml"
                            sed 's/127.0.0.1/host.docker.internal/g' "\$ORIG_KUBE" > "\$PATCHED"
                            export KUBECONFIG="\$PATCHED"
                            KUBE_EXTRA="-e kubeconfig=\$PATCHED"
                            echo "Kubeconfig adapté pour Docker (127.0.0.1 -> host.docker.internal), passé à Ansible via -e kubeconfig."
                        fi
                        ansible --version
                        TAGS=""
                        case "${params.ANSIBLE_TAGS}" in
                            install)      TAGS="--tags install" ;;
                            applications) TAGS="--tags applications" ;;
                        esac
                        ansible-playbook playbooks/gitops.yml \$TAGS \$KUBE_EXTRA -v
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ GitOps Ansible terminé (ArgoCD + applications)."
        }
        failure {
            echo "❌ Échec du playbook Ansible. Vérifier KUBECONFIG, contexte kubectl et group_vars/all.yml."
        }
    }
}
