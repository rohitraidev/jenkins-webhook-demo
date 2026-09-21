pipeline {
    agent {
        label 'control-built-in'
    }

 

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Get Server IP') {
            steps {
                script {
                    env.SERVER_IP = sh(
                        script: "hostname -I | awk '{print \$1}'",
                        returnStdout: true
                    ).trim()

                    echo "Server IP: ${env.SERVER_IP}"
                }
            }
        }

        stage('Deploy Website') {
            steps {
                sh '''
                    echo "Deploying website..."
                    echo "Running as user: $(whoami)"
                    echo "Hostname: $(hostname)"

                    rm -rf /var/www/html/*
                    cp index.html /var/www/html/

                    echo "Website deployed successfully"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Checking website..."

                    curl -f http://localhost

                    echo ""
                    echo "Website is working!"
                '''
            }
        }
    }

    post {
        success {
            echo 'Website deployment successful!'
        }

        failure {
            echo 'Website deployment failed!'
        }
    }
}
