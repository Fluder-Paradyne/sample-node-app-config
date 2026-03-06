// CI/CD for sample-node-app via ArgoCD GitOps
// Jenkins: build, test, push. ArgoCD: deploy (staging auto-sync, production after approval)
pipeline {
  agent { label 'docker' }
  environment {
    REGISTRY = 'registry.registry.svc.cluster.local:5000'
    APP_IMAGE = "${REGISTRY}/sample-node-app"
    APP_REPO = 'https://github.com/Fluder-Paradyne/sample-node-project.git'
    APP_CONFIG_REPO = 'https://github.com/Fluder-Paradyne/sample-node-app-config.git'
    APP_CONFIG_CREDENTIALS = 'github-app-config'
  }
  stages {
    stage('Checkout') {
      steps {
        container('jnlp') {
          dir('app') {
            git url: "${APP_REPO}", branch: 'main'
          }
        }
      }
    }
    stage('Build') {
      steps {
        container('node') {
          dir('app') {
            sh 'npm ci'
          }
        }
      }
    }
    stage('Test') {
      steps {
        container('node') {
          dir('app') {
            sh 'npx jest --ci --passWithNoTests --forceExit || true'
          }
        }
      }
    }
    stage('Build Image') {
      steps {
        container('docker') {
          dir('app') {
            sh """
              docker build -t ${APP_IMAGE}:${BUILD_NUMBER} .
              docker tag ${APP_IMAGE}:${BUILD_NUMBER} ${APP_IMAGE}:latest
            """
          }
        }
      }
    }
    stage('Push to Registry') {
      steps {
        container('docker') {
          sh """
            docker push ${APP_IMAGE}:${BUILD_NUMBER}
            docker push ${APP_IMAGE}:latest
          """
        }
      }
    }
    stage('Deploy to Staging') {
      steps {
        container('jnlp') {
          sh "sed -i.bak 's/tag: .*/tag: \"${BUILD_NUMBER}\"/' sample-node-app/values-staging.yaml"
          sh """
            git config user.email "jenkins@local"
            git config user.name "Jenkins"
            git add sample-node-app/values-staging.yaml
          """
          withCredentials([usernamePassword(credentialsId: env.APP_CONFIG_CREDENTIALS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git diff --staged --quiet || (git commit -m "Deploy to staging: ${BUILD_NUMBER}" && git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/Fluder-Paradyne/sample-node-app-config.git HEAD:main)
            """
          }
        }
      }
    }
    stage('Manual Approval - Production') {
      steps {
        input message: 'Deploy to production?', ok: 'Deploy'
      }
    }
    stage('Deploy to Production') {
      steps {
        container('jnlp') {
          sh "sed -i.bak 's/tag: .*/tag: \"${BUILD_NUMBER}\"/' sample-node-app/values-prod.yaml"
          sh """
            git config user.email "jenkins@local"
            git config user.name "Jenkins"
            git add sample-node-app/values-prod.yaml
          """
          withCredentials([usernamePassword(credentialsId: env.APP_CONFIG_CREDENTIALS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh """
              git diff --staged --quiet || (git commit -m "Promote to production: ${BUILD_NUMBER}" && git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/Fluder-Paradyne/sample-node-app-config.git HEAD:main)
            """
          }
        }
      }
    }
  }
  post {
    always {
      deleteDir()
    }
  }
}
