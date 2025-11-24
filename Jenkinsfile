
pipeline {
    agent {
        docker {
            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
        }
    }

    environment {
        NODE_ENV = 'test'
    }

    stages {
        stage('Build') {
            steps {
                sh '''
                    node --version
                    npm --version
                    npm ci
                    npm run build
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    test -f build/index.html
                    npm test
                '''
            }
        }

        stage('E2E') {
            steps {
                sh '''
                    # Install compatible wait-on version for Node 18
                    npm install serve wait-on@6.0.0

                    # Ensure build output exists
                    test -f build/index.html

                    # Start server in background
                    node_modules/.bin/serve -s build &amp;amp;

                    # Wait for index.html to be served
                    npx wait-on http-get://localhost:3000/index.html

                    # Run Playwright tests (HTML report)
                    npx playwright test --reporter=html
                '''
            }
        }

        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                '''
            }
        }
    }

    post {
        always {
            script {
                // Ensure a workspace/launcher exists for post steps
                node(env.NODE_NAME) {
                    // Do not fail the build if the JUnit XML isn't present
                    junit allowEmptyResults: true, testResults: 'jest-results/junit.xml'

                    // Archive the Playwright HTML report directory (does not require extra plugins)
                    archiveArtifacts artifacts: 'playwright-report/**', allowEmptyArchive: true
                }
            }
        }
    }
}

