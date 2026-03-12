pipeline {
  agent any

  environment {
    DOCKER_USER = "darninidhi2122"
    DOCKER_CRED = "dockerhub-creds"

    FRONTEND_IMAGE = "kubecoin-frontend"
    BACKEND_IMAGE  = "kubecoin-backend"

    IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
  }

  triggers {
    githubPush()
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set Environment Namespace') {
      steps {
        script {
          if (env.BRANCH_NAME == 'prod') {
            env.K8S_NAMESPACE = 'production'
          } else if (env.BRANCH_NAME == 'dev') {
            env.K8S_NAMESPACE = 'dev'
          } else if (env.BRANCH_NAME == 'test') {
            env.K8S_NAMESPACE = 'testing'
          } else {
            error "Unsupported branch: ${env.BRANCH_NAME}"
          }
        }
      }
    }

    stage('Docker Login') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: DOCKER_CRED,
          usernameVariable: 'DOCKER_USERNAME',
          passwordVariable: 'DOCKER_PASSWORD'
        )]) {
          sh '''
            echo "$DOCKER_PASSWORD" | docker login \
              -u "$DOCKER_USERNAME" --password-stdin
          '''
        }
      }
    }

    stage('Build Docker Images') {
      steps {
        sh """
          docker build -t $DOCKER_USER/$FRONTEND_IMAGE:$IMAGE_TAG frontend/
          docker build -t $DOCKER_USER/$BACKEND_IMAGE:$IMAGE_TAG backend/
        """
      }
    }

    stage('Push Docker Images') {
      steps {
        sh """
          docker push $DOCKER_USER/$FRONTEND_IMAGE:$IMAGE_TAG
          docker push $DOCKER_USER/$BACKEND_IMAGE:$IMAGE_TAG
        """
      }
    }

    stage('Approve Production') {
      when { branch 'prod' }
      steps {
        input message: "Approve deployment of ${IMAGE_TAG} to PRODUCTION?"
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh """
        kubectl apply -f assignemnt/${K8S_NAMESPACE}/

      kubectl set image deployment/frontend \
        frontend=$DOCKER_USER/kubecoin-frontend:$IMAGE_TAG \
        -n ${K8S_NAMESPACE}

      kubectl set image deployment/backend \
        backend=$DOCKER_USER/kubecoin-backend:$IMAGE_TAG \
        -n ${K8S_NAMESPACE}

      kubectl rollout status deployment/frontend -n ${K8S_NAMESPACE}
      kubectl rollout status deployment/backend -n ${K8S_NAMESPACE}
        """
      }
    }
  }
}
