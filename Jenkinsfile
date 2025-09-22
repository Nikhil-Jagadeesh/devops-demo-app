pipeline {
  agent any
  environment {
    DOCKERHUB_CRED = credentials('dockerhub-creds')  // set in Jenkins
    DOCKERHUB_NS   = 'DOCKER_HUB_USERNAME'           // change me
    IMAGE_NAME     = "${DOCKERHUB_NS}/hello-minikube"
  }
  triggers { githubPush() }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.GIT_COMMIT_SHORT = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('MySonar') {
          sh """
            if ! command -v sonar-scanner >/dev/null 2>&1; then
              curl -sSLo ss.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-5.0.1.3006-linux.zip
              unzip -q -o ss.zip
              export PATH=\$PWD/sonar-scanner-*/bin:\$PATH
            fi
            sonar-scanner
          """
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${IMAGE_NAME}:${env.GIT_COMMIT_SHORT} ."
      }
    }

    stage('Push Image') {
      steps {
        sh """
          echo '${DOCKERHUB_CRED_PSW}' | docker login -u '${DOCKERHUB_CRED_USR}' --password-stdin
          docker tag ${IMAGE_NAME}:${env.GIT_COMMIT_SHORT} ${IMAGE_NAME}:latest
          docker push ${IMAGE_NAME}:${env.GIT_COMMIT_SHORT}
          docker push ${IMAGE_NAME}:latest
        """
      }
    }

    stage('Deploy to Minikube') {
      steps {
        sh """
          kubectl config use-context minikube
          sed -e 's/DOCKER_HUB_USERNAME/${DOCKERHUB_NS}/g' \
              -e 's/\\${GIT_COMMIT_SHORT}/${GIT_COMMIT_SHORT}/g' \
              k8s/deployment.yaml | kubectl apply -f -
          kubectl rollout status deploy/hello-deploy --timeout=90s
        """
      }
    }
  }
}
