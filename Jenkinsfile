
pipeline {
    agent any

    options {
        // keep console readable and fail fast on silent errors
        ansiColor('xterm')
        timestamps()
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    // Reuse the same workspace on the Jenkins node
                    reuseNode true
                    // Optional: enable Docker socket mount if you need Docker inside the container
                    // args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh '''
                    set -euxo pipefail
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la build || true
                '''
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -euxo pipefail
                    # basic artifact sanity check
                    test -f build/index.html
                    npm test -- --ci
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit/*.xml'
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'build/**/*', allowEmptyArchive: true
        }
    }
}
