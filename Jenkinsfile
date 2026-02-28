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
        choice(
            name: 'ANSIBLE_TAGS',
            choices: ['all', 'install', 'applications'],
            description: 'Tags Ansible : all = playbook complet, install = ArgoCD uniquement, applications = Application CRs uniquement'
        )
    }

    environment {
        ANSIBLE_DIR = 'infra/ansible'
        // KUBECONFIG utilisé par Ansible (agent : volume monté /home/jenkins/.kube)
        KUBECONFIG = "${env.HOME}/.kube/config"
    }

    stages {
        stage('📥 Checkout') {
            steps {
                checkout scm
                sh 'ls -la infra/ansible/ 2>/dev/null || (echo "Erreur: infra/ansible/ introuvable (vérifier le repo du job)" && exit 1)'
            }
        }

        stage('🔧 Ansible') {
            steps {
                dir(env.ANSIBLE_DIR) {
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
