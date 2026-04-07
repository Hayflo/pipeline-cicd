pipeline {
    agent any
    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        WEB_SERVER   = '192.168.56.11'
        DB_SERVER    = '192.168.56.12'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Branche : ${env.GIT_BRANCH}"
                echo "Commit  : ${env.GIT_COMMIT}"
            }
        }
        stage('Vérification Ansible') {
            steps {
                sh 'ansible --version'
                sh 'ansible-playbook --syntax-check -i ansible/hosts ansible/playbook.yml'
            }
        }
        stage('Test connectivité VMs') {
            steps {
                sh 'ansible all -i ansible/hosts -m ping'
            }
        }
        stage('Déploiement Ansible') {
            steps {
                sh '''
                ansible-playbook \
                  -i ansible/hosts \
                  ansible/playbook.yml \
                  -v
                '''
            }
        }
        stage('Vérification déploiement') {
            steps {
                sh "curl -f http://${WEB_SERVER} || exit 1"
                echo 'Serveur web répond correctement'
            }
        }
    }
    post {
        success {
            echo "Déploiement réussi sur ${env.GIT_BRANCH}"
        }
        failure {
            echo 'Échec — lancement du rollback Ansible'
            sh '''
            ansible-playbook \
              -i ansible/hosts \
              ansible/rollback.yml
            '''
        }
        always {
            echo "Pipeline terminé — statut : ${currentBuild.currentResult}"
        }
    }
}
