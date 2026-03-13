pipeline {

agent {
  kubernetes {
    label 'devops-agent'
    defaultContainer 'devops'
  }
}

environment {
  DOCKER_USER = "darninidhi2122"
  DOCKER_CRED = "dockerhub-creds"

  FRONTEND_IMAGE = "kubecoin-frontend"
  BACKEND_IMAGE  = "kubecoin-backend"

  IMAGE_TAG = "${env.BUILD_NUMBER}"

  APP_NAMESPACE   = "app"
  INFRA_NAMESPACE = "infra"

  APP_CHART   = "./kubecoin-chart/app"
  INFRA_CHART = "./kubecoin-chart/postgres"
}

stages {

stage('Checkout Source') {
  steps {
    checkout scm
  }
}

stage('Verify Tools') {
  steps {
    sh '''
    docker --version
    kubectl version --client
    helm version
    '''
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

stage('Build and Push Images') {
  steps {
    sh """
    /kaniko/executor \
      --context=frontend \
      --dockerfile=frontend/Dockerfile \
      --destination=$DOCKER_USER/$FRONTEND_IMAGE:$IMAGE_TAG

    /kaniko/executor \
      --context=backend \
      --dockerfile=backend/Dockerfile \
      --destination=$DOCKER_USER/$BACKEND_IMAGE:$IMAGE_TAG
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

stage('Create Namespaces') {
  steps {
    sh '''
    kubectl get ns $APP_NAMESPACE || kubectl create ns $APP_NAMESPACE
    kubectl get ns $INFRA_NAMESPACE || kubectl create ns $INFRA_NAMESPACE
    '''
  }
}

stage('Deploy Postgres') {
  steps {
    sh """
    helm upgrade --install postgres ${INFRA_CHART} \
      --namespace ${INFRA_NAMESPACE} \
      --create-namespace
    """
  }
}

stage('Deploy App') {
  steps {
    sh """
    helm upgrade --install kubecoin ${APP_CHART} \
      --namespace ${APP_NAMESPACE} \
      --set frontend.image=$DOCKER_USER/$FRONTEND_IMAGE \
      --set frontend.tag=$IMAGE_TAG \
      --set backend.image=$DOCKER_USER/$BACKEND_IMAGE \
      --set backend.tag=$IMAGE_TAG
    """
  }
}

stage('Verify Deployment') {
  steps {
    sh '''
    kubectl get pods -n app
    kubectl get svc -n app
    kubectl get pods -n infra
    kubectl get svc -n infra
    '''
  }
}

}

post {
  success {
    echo "Application deployed successfully using Helm."
  }
  failure {
    echo "Pipeline failed. Check logs."
  }
  always {
    cleanWs()
  }
}

}
