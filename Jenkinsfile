
pipeline {
    agent any

    environment {
        APP_NAME = 'my-node-app'
        VERSION  = "${env.BUILD_NUMBER}"
        IMAGE    = "registry.example.com/${env.APP_NAME}:${env.VERSION}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Unit Tests (Node in Docker)') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -euxo pipefail
                    npm ci
                    npm test -- --ci
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit/*.xml'
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    // Build a Docker image using the Docker Pipeline API
                    def appImage = docker.build(env.IMAGE)
                }
            }
        }

        stage('Smoke Test Image') {
            steps {
                script {
                    docker.withRegistry('', 'registry-credentials-id') {
                        // Run the image temporarily to smoke test
                        sh """
                            set -euxo pipefail
                            cid=\$(docker run -d --rm -p 8080:8080 ${env.IMAGE})
                            trap 'docker rm -f \$cid || true' EXIT
                            # add your health check here (curl or node script)
                            sleep 5
                            docker logs \$cid
                        """
                    }
                }
            }
        }

        stage('Push Image') {
            when {
                expression { return env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master' }
            }
            steps {
                script {
                    docker.withRegistry('https://registry.example.com', 'registry-credentials-id') {
                        sh "docker push ${env.IMAGE}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Image pushed: ${env.IMAGE}"
        }
    }
}

