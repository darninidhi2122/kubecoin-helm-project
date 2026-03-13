pipeline {

agent {
  kubernetes {
    inheritFrom 'devops-agent'
    defaultContainer 'devops'
  }
}

environment {

  DOCKER_USER = "darninidhi2122"

  FRONTEND_IMAGE = "kubecoin-frontend"
  BACKEND_IMAGE  = "kubecoin-backend"

  IMAGE_TAG = "${env.BUILD_NUMBER}"

  APP_NAMESPACE   = "app"
  INFRA_NAMESPACE = "infra"

  APP_CHART   = "./kubecoin-chart/app"
  INFRA_CHART = "./kubecoin-chart/infra"
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
    kubectl version --client
    helm version
    '''
  }
}

stage('Build & Push Images (Kaniko)') {
  steps {
    container('kaniko') {
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
}

stage('Create Namespaces') {
  steps {
    sh '''
    kubectl get ns $APP_NAMESPACE || kubectl create ns $APP_NAMESPACE
    kubectl get ns $INFRA_NAMESPACE || kubectl create ns $INFRA_NAMESPACE
    '''
  }
}

stage('Deploy Postgres (Infra)') {
  steps {
    sh """
    helm upgrade --install postgres ${INFRA_CHART} \
      --namespace ${INFRA_NAMESPACE} \
      --create-namespace
    """
  }
}

stage('Deploy Application (App Namespace)') {
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
    echo "Checking app namespace..."
    kubectl get pods -n $APP_NAMESPACE
    kubectl get svc -n $APP_NAMESPACE

    echo "Checking infra namespace..."
    kubectl get pods -n $INFRA_NAMESPACE
    kubectl get svc -n $INFRA_NAMESPACE
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
