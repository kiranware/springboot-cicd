pipeline {
  agent any

  environment {
    DOCKERHUB_USER = "kiranwaredockerpoc"
    APP_NAME       = "springboot-cicd"
    IMAGE_REPO     = "${DOCKERHUB_USER}/${APP_NAME}"

    MANIFESTS_REPO_URL = "https://github.com/kiranware/springboot-cicd-manifests.git"
    MANIFESTS_REPO_DIR = "manifests-repo"

    DOCKERHUB_CREDS_ID      = "dockerhub-creds"
    GITHUB_TOKEN_CREDS_ID   = "github-token"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    // ✅ Fix mvn not found: run build inside Maven container
    stage('Build & Test') {
      agent {
        docker {
          image 'maven:3.9.9-eclipse-temurin-17'
          args  '-v /root/.m2:/root/.m2'
        }
      }
      steps {
        sh 'mvn -B clean test'
      }
    }

    // ✅ Build image using Docker CLI container + host docker socket
    stage('Build Docker Image') {
      agent {
        docker {
          image 'docker:27-cli'
          args  '-v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        script {
          env.IMAGE_TAG = "${BUILD_NUMBER}"
          sh "docker build -t ${IMAGE_REPO}:${IMAGE_TAG} ."
        }
      }
    }

    stage('Push Image to DockerHub') {
      agent {
        docker {
          image 'docker:27-cli'
          args  '-v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDS_ID, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh """
            echo "\$DH_PASS" | docker login -u "\$DH_USER" --password-stdin
            docker push ${IMAGE_REPO}:${IMAGE_TAG}
          """
        }
      }
    }

    // ✅ Update manifests using git + sed container
    stage('Update K8s Manifests Repo') {
      agent {
        docker { image 'alpine:3.20' }
      }
      steps {
        withCredentials([string(credentialsId: GITHUB_TOKEN_CREDS_ID, variable: 'GITHUB_TOKEN')]) {
          sh """
            apk add --no-cache git sed

            rm -rf ${MANIFESTS_REPO_DIR}
            git clone https://x-access-token:\$GITHUB_TOKEN@github.com/kiranware/springboot-cicd-manifests.git ${MANIFESTS_REPO_DIR}
            cd ${MANIFESTS_REPO_DIR}

            sed -i "s#image: ${IMAGE_REPO}:.*#image: ${IMAGE_REPO}:${IMAGE_TAG}#" k8s/deployment.yaml

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
