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
            name: 'ANSIBLE_DIR',
            defaultValue: 'infra/ansible',
            description: 'Chemin vers le répertoire Ansible (playbooks + roles). Par défaut infra/ansible. Si votre repo a ansible à la racine, mettre ansible.'
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
                sh """
                    ANSIBLE_DIR='${params.ANSIBLE_DIR}'
                    if [ ! -d "\$ANSIBLE_DIR" ] || [ ! -f "\$ANSIBLE_DIR/playbooks/gitops.yml" ]; then
                        echo "Erreur: \$ANSIBLE_DIR/ introuvable ou playbooks/gitops.yml manquant."
                        echo "Le repo cloné doit contenir le dossier Ansible (ex. infra/ansible/ avec playbooks/ et roles/)."
                        echo "Soit : configurer le job pour cloner un repo qui a infra/ à la racine."
                        echo "Soit : utiliser le paramètre ANSIBLE_DIR (ex. ansible si le dossier est à la racine)."
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
