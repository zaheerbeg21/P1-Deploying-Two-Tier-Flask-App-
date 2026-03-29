pipeline {
    agent any

    environment {
        APP_SERVER = 'ubuntu@172.31.44.4'   // App Server private IP
        APP_DIR    = '/home/ubuntu/app'
    }

    stages {
        stage('Clone Repo') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/zaheerbeg21/P1-Deploying-Two-Tier-Flask-App-.git'
            }
        }

        stage('Copy Files to App Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no \
                            docker-compose.yml \
                            Dockerfile \
                            app.py \
                            requirement.txt \
                            $APP_SERVER:$APP_DIR/
                    '''
                }
            }
        }

        stage('Deploy on App Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no $APP_SERVER "
                            cd $APP_DIR &&
                            docker compose down || true &&
                            docker compose up -d --build
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful! Flask app is live at http://65.0.31.38:5000'
        }
        failure {
            echo '❌ Pipeline failed. Check Console Output for details.'
        }
    }
}
