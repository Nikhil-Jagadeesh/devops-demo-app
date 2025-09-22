pipeline {
  agent any
  environment {
    DOCKERHUB_CRED = credentials('dockerhub-creds')  // Jenkins credentials ID
    DOCKERHUB_NS   = 'alphacoder2019'                // <-- change to your Docker Hub username
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
          script {
            def scannerHome = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
            sh """
              "${scannerHome}/bin/sonar-scanner" \
                -Dsonar.projectKey=hello-minikube \
                -Dsonar.sources=. \
                -Dsonar.host.url=${SONAR_HOST_URL} \
                -Dsonar.login=${SONAR_AUTH_TOKEN}
            """
          }
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
