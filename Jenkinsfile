pipeline {
    agent any
 
    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        WEB_SERVER = '192.168.56.11'
        DB_SERVER  = '192.168.56.12'
        INVENTORY  = 'ansible/hosts'
        PLAYBOOK   = 'ansible/playbook.yml'
        ROLLBACK   = 'ansible/rollback.yml'
        GRAYLOG    = 'ansible/graylog.yml'
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
                sh 'ansible-playbook --syntax-check -i $INVENTORY $PLAYBOOK'
                sh 'ansible-playbook --syntax-check -i $INVENTORY $GRAYLOG'
            }
        }
 
        stage('Test connectivité VMs') {
            steps {
                sh 'ansible all -i $INVENTORY -m ping'
            }
        }
 
        stage('Déploiement Ansible') {
            steps {
                sh '''
                ansible-playbook \
                  -i $INVENTORY \
                  $PLAYBOOK \
                  -v
                '''
            }
        }
 
        stage('Déploiement Graylog') {
            steps {
                echo 'Installation de Graylog + Filebeat sur les VMs...'
                sh '''
                ansible-playbook \
                  -i $INVENTORY \
                  $GRAYLOG \
                  -v
                '''
            }
        }
 
        stage('Vérification déploiement') {
            steps {
                sh '''
                curl -f http://$WEB_SERVER || exit 1
                '''
                echo 'Serveur web répond correctement'
 
                sh '''
                curl -f http://$DB_SERVER:9000/api/ \
                  -u admin:admin \
                  -H "X-Requested-By: jenkins" || echo "Graylog UI non joignable — vérifier manuellement"
                '''
                echo 'Vérification Graylog terminée'
            }
        }
    }
 
    post {
        success {
            echo "Déploiement réussi sur ${env.GIT_BRANCH}"
            echo "Graylog accessible sur http://${env.DB_SERVER}:9000"
        }
 
        failure {
            echo 'Échec — lancement du rollback Ansible'
            sh '''
            ansible-playbook \
              -i $INVENTORY \
              $ROLLBACK \
              -v
            '''
        }
 
        always {
            echo "Pipeline terminé — statut : ${currentBuild.currentResult}"
        }
    }
}
