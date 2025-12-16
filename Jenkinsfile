pipeline {
  agent any

  environment {
    DOCKERHUB_USER = "kiranwaredockerpoc"
    APP_NAME = "springboot-cicd"
    IMAGE_REPO = "${DOCKERHUB_USER}/${APP_NAME}"
    IMAGE_TAG = "${BUILD_NUMBER}"

    MANIFESTS_REPO_DIR = "manifests-repo"

    DOCKERHUB_CREDS_ID = "dockerhub-creds"
    GITHUB_TOKEN_CREDS_ID = "github-token"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps {
        sh "chmod +x mvnw || true"
        sh "./mvnw -B clean test"
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${IMAGE_REPO}:${IMAGE_TAG} ."
      }
    }

    stage('Push Image to DockerHub') {
      steps {
        withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDS_ID, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh """
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push ${IMAGE_REPO}:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Update K8s Manifests Repo') {
      steps {
        withCredentials([string(credentialsId: GITHUB_TOKEN_CREDS_ID, variable: 'GITHUB_TOKEN')]) {
          sh """
            rm -rf ${MANIFESTS_REPO_DIR}
            git clone https://$GITHUB_TOKEN@github.com/kiranware/springboot-cicd-manifests.git ${MANIFESTS_REPO_DIR}
            cd ${MANIFESTS_REPO_DIR}

            sed -i 's#image: .*#image: ${IMAGE_REPO}:${IMAGE_TAG}#' k8s/deployment.yaml

            git config user.email "jenkins@local"
            git config user.name "jenkins-bot"
            git add k8s/deployment.yaml
            git commit -m "Update image to ${IMAGE_REPO}:${IMAGE_TAG}" || echo "No changes"
            git push
          """
        }
      }
    }
  }
}
