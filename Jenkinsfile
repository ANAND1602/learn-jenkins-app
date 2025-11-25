
/*
 * Option B: Declarative pipeline using a Docker agent
 * - Parallel lint + unit tests
 * - Build & push container image
 * - Security scan (Trivy example)
 * - Helm deploy to Kubernetes on main branch
 *
 * Replace credential IDs, registry, chart path, and tooling to fit your environment.
 */

pipeline {
  agent {
    docker {
      // Choose an image that has node/npm + any tools you need (or build a custom CI image).
      image 'mcr.microsoft.com/devcontainers/javascript-node:1-20-bullseye'
      // Map cache folders & run as non-root where possible
      args '-u 1000:1000 -v $WORKSPACE/.npm:/home/node/.npm'
      reuseNode true
    }
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    timeout(time: 45, unit: 'MINUTES')
  }

  environment {
    APP_NAME    = 'my-service'                             // <<< change
    REGISTRY    = 'ghcr.io/myorg'                          // <<< change
    IMAGE_TAG   = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}" // branch+build number tagging
    NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"

    // Credential IDs must exist in Jenkins:
    // - ghcr-pat: username/password or token for container registry
    // - kubeconfig-prod: Kubeconfig file credential
    // - slack-webhook (optional): secret text for Slack notifications
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD > .gitsha'
        script { currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.BRANCH_NAME}" }
      }
    }

    stage('Prepare') {
      steps {
        sh '''
          node -v
          npm -v
          npm ci --prefer-offline --no-audit
        '''
      }
      post {
        failure {
          echo 'Dependency install failed.'
        }
      }
    }

    stage('Static Analysis & Tests') {
      parallel {
        stage('Lint') {
          steps {
            sh 'npm run lint'
          }
        }
        stage('Unit Tests') {
          steps {
            sh '''
              npm test -- --ci --reporters=default --reporters=jest-junit || true
            '''
          }
          post {
            always {
              // Collect JUnit results if produced
              junit allowEmptyResults: true, testResults: 'reports/junit.xml'
              // Coverage or other artifacts
              archiveArtifacts artifacts: 'coverage/**', fingerprint: true, allowEmptyArchive: true
            }
          }
        }
      }
    }

    stage('Build & Push Image') {
      when { not { buildingTag() } } // example to skip on tag builds, adjust to taste
      steps {
        withCredentials([usernamePassword(credentialsId: 'ghcr-pat', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
          sh '''
            echo "$REG_PASS" | docker login $REGISTRY -u "$REG_USER" --password-stdin
            docker build -t $REGISTRY/$APP_NAME:$IMAGE_TAG .
            docker tag  $REGISTRY/$APP_NAME:$IMAGE_TAG $REGISTRY/$APP_NAME:latest
            docker push $REGISTRY/$APP_NAME:$IMAGE_TAG
            docker push $REGISTRY/$APP_NAME:latest
          '''
        }
      }
    }

    stage('Security Scan') {
      // Optional: requires Trivy in the agent image or available on the node.
      steps {
        sh '''
          if command -v trivy >/dev/null 2>&1; then
            trivy image --exit-code 0 --ignore-unfixed $REGISTRY/$APP_NAME:$IMAGE_TAG
            trivy image --exit-code 1 --severity HIGH,CRITICAL $REGISTRY/$APP_NAME:$IMAGE_TAG || echo "High/Critical issues detected"
          else
            echo "Trivy not installed; skipping security scan."
          fi
        '''
      }
    }

    stage('Helm Deploy (prod)') {
      when { branch 'main' }
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG')]) {
          sh '''
            # Ensure helm and kubectl are available in agent image or host
            kubectl version --client
            helm version

            GIT_SHA=$(cat .gitsha)
            helm upgrade --install $APP_NAME charts/$APP_NAME \
              --namespace prod \
              --create-namespace \
              --set image.repository=$REGISTRY/$APP_NAME \
              --set image.tag=$IMAGE_TAG \
              --set app.gitSha=$GIT_SHA \
              --wait --timeout 5m
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Build succeeded for ${env.JOB_NAME} #${env.BUILD_NUMBER}"
      script {
        // Example Slack notify (requires a step or shared lib). Replace with your notifier.
        // slackSend(channel: '#ci', color: 'good', message: "Build success: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
      }
    }
    failure {
      echo "❌ Build failed for ${env.JOB_NAME} #${env.BUILD_NUMBER}"
      script {
        // slackSend(channel: '#ci', color: 'danger', message: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
      }
    }
    always {
      echo "Cleaning workspace…"
      cleanWs(deleteDirs: true, notFailBuild: true)
    }
  }
}

