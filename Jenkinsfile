pipeline {
agent any

environment {
DOCKER_USER = "darninidhi2122"
DOCKER_CRED = "dockerhub-creds"

```
FRONTEND_IMAGE = "kubecoin-frontend"
BACKEND_IMAGE  = "kubecoin-backend"

IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

APP_NAMESPACE   = "app"
INFRA_NAMESPACE = "infra"

APP_CHART   = "./kubecoin-chart/app"
INFRA_CHART = "./kubecoin-chart/postgres"
```

}

triggers {
githubPush()
}

stages {

```
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

stage('Create Namespaces if Missing') {
  steps {
    sh '''
    kubectl get ns $APP_NAMESPACE || kubectl create ns $APP_NAMESPACE
    kubectl get ns $INFRA_NAMESPACE || kubectl create ns $INFRA_NAMESPACE
    '''
  }
}

stage('Deploy Infrastructure (Postgres)') {
  steps {
    sh """
    helm upgrade --install postgres ${INFRA_CHART} \
      --namespace ${INFRA_NAMESPACE} \
      --create-namespace
    """
  }
}

stage('Deploy Application (Frontend + Backend)') {
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
    kubectl get pods -n app
    kubectl get svc -n app

    echo "Checking infra namespace..."
    kubectl get pods -n infra
    kubectl get svc -n infra
    '''
  }
}
```

}

post {

```
success {
  echo "Application deployed successfully using Helm."
}

failure {
  echo "Pipeline failed. Check logs."
}

always {
  cleanWs()
}
```

}
}
