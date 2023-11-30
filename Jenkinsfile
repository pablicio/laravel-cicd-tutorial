pipeline {

    agent any

    stages {
        stage("Verify tooling") {
            steps {
                sh '''
                docker info
                docker version
                docker-compose version
                '''
            }
        }

        stage("Verify SSH connection to server") {
            steps {
                sshagent(credentials: ['laravel-jenkins']) {
                    sh '''
                    ssh -o StrictHostKeyChecking=no vagrant@192.168.33.10 whoami
                    '''
                }
            }
        }

        stage("Clear all running docker containers") {
            steps {
                script {
                    try {
                        sh 'docker rm -f $(docker ps -a -q)'
                    } catch (Exception e) {
                        echo 'No running container to clear up...'
                    }
                }
            }
        }

        stage("Start Docker") {
            steps {
                sh 'make up'
                sh 'docker-compose ps'
            }
        }

        stage("Run Composer Install") {
            steps {
                sh 'docker-compose run --rm composer install'
            }
        }

        stage('Carregando o ENV de desenvolvimento') {
            steps {
                configFileProvider([configFile(fileId: '71e14e4c-3c32-473d-8284-3334c22dbe7f', variable: 'env')]) {
                  echo '$env'
                  sh 'ssh -o StrictHostKeyChecking=no vagrant@192.168.33.10 cat $env > /var/lib/jenkins/workspace/laravel-jenkins/.env'
                }
            }
        }

        stage("Run Tests") {
            steps {
                sh 'docker-compose run --rm artisan test'
            }
        }
    }

    post {
        success {
            sh 'cd "/var/lib/jenkins/workspace/laravel-jenkins"'
            sh 'rm -rf artifact.zip'
            sh 'zip -r artifact.zip . -x "*node_modules**"'

            withCredentials([sshUserPrivateKey(credentialsId: "laravel-jenkins", keyFileVariable: 'keyfile')]) {
                sh 'scp -v -o StrictHostKeyChecking=no -i ${keyfile} /var/lib/jenkins/workspace/laravel-jenkins/artifact.zip vagrant@192.168.33.10:/vagrant/artifact'
            }

            sshagent(credentials: ['laravel-jenkins']) {
                sh 'ssh -o StrictHostKeyChecking=no vagrant@192.168.33.10 unzip -o /vagrant/artifact/artifact.zip -d /var/www/html'
                script {
                    try {
                        sh 'ssh -o StrictHostKeyChecking=no vagrant@192.168.33.10 sudo chmod 777 /var/www/html/storage -R'
                    } catch (Exception e) {
                        echo 'Some file permissions could not be updated.'
                    }
                }
            }
        }

        always {
            sh 'docker-compose down --remove-orphans -v'
            sh 'docker-compose ps'
        }
    }
}