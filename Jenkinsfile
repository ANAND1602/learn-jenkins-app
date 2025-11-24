
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

                    # Run Playwright tests
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
            // Allocate a workspace/launcher for post steps
            node {
                junit 'jest-results/junit.xml'
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: false,
                    keepAll: false,
                    reportDir: 'playwright-report',
                    reportFiles: 'index.html',
                    reportName: 'Playwright HTML Report',
                    reportTitles: '',
                    useWrapperFileDirectly: true
                ])
            }
        }
    }
}

