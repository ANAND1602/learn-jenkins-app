
pipeline {
    agent none

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    // Keeps the same Jenkins node (not container), fine for mounted workspace
                    reuseNode true
                    // Uncomment if native builds are needed (e.g., node-gyp):
                    // args '-u root:root'
                }
            }
            environment {
                // Prevents lifecycle scripts from running if your project uses them unintentionally
                // NPM_CONFIG_LOGLEVEL = 'warn'
                // NODE_ENV = 'production'
            }
            steps {
                sh '''
                    set -euxo pipefail

                    node --version
                    npm --version

                    # If you need native deps for npm builds, uncomment:
                    # apk add --no-cache python3 make g++ git

                    ls -la

                    # Clean install based on package-lock.json
                    npm ci --no-audit --no-fund

                    # Build the project
                    npm run build

                    ls -la build || true
                '''
            }
            post {
                success {
                    // Keep build artifacts for debugging or deployment steps
                    archiveArtifacts artifacts: 'build/**', onlyIfSuccessful: true
                }
                always {
                    // If you installed build deps, you can clean them here if needed
                    sh 'echo "Build stage done."'
                }
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                    // args '-u root:root'
                }
            }
            steps {
                sh '''
                    set -euxo pipefail

                    node --version
                    npm --version

                    # Ensure test dependencies are present
                    npm ci --no-audit --no-fund

                    # Verify build artifact exists from previous stage
                    test -f build/index.html

                    # Run tests (adjust flags as needed)
                    npm test -- --ci
                '''
            }
            post {
                always {
                    // If you produce JUnit or coverage reports, publish them:
                    // junit 'reports/junit/*.xml'
                    // publishHTML(target: [reportDir: 'coverage', reportFiles: 'index.html', reportName: 'Coverage'])
                    sh 'echo "Test stage done."'
                }
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed.'
        }
        success {
            echo 'Pipeline succeeded.'
        }
        always {
            // General cleanup hook
            echo 'Pipeline finished.'
        }
    }
