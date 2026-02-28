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
        KUBECONFIG = "${env.HOME}/.kube/config"
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
                }
                sh """
                    ANSIBLE_DIR='${params.ANSIBLE_DIR}'
                    if [ ! -d "\$ANSIBLE_DIR" ] || [ ! -f "\$ANSIBLE_DIR/playbooks/gitops.yml" ]; then
                        echo "Erreur: \$ANSIBLE_DIR/ introuvable ou playbooks/gitops.yml manquant."
                        echo "  - Configurer le job pour cloner un repo qui a infra/ à la racine, OU"
                        echo "  - Renseigner le paramètre INFRA_REPO_URL (URL du repo contenant ansible/ et argocd/ à la racine)."
                        exit 1
                    fi
                    ls -la "\$ANSIBLE_DIR/"
                """
            }
        }

        stage('🔧 Ansible') {
            steps {
                dir("${params.ANSIBLE_DIR}") {
                    sh """
                        set -e
                        ansible --version
                        TAGS=""
                        case "${params.ANSIBLE_TAGS}" in
                            install)      TAGS="--tags install" ;;
                            applications) TAGS="--tags applications" ;;
                        esac
                        ansible-playbook playbooks/gitops.yml \$TAGS -v
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
