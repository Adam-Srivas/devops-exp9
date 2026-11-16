pipeline {
  agent any
  environment {
    IMAGE = "adamyasrivastava/healthsite"
    TAG   = "${BUILD_NUMBER}"
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Secret Scan (Gitleaks)') {
      steps {
        sh 'docker run --rm -v $PWD:/repo zricethezav/gitleaks:latest detect --source=/repo --no-git --verbose'
      }
    }
    stage('Dockerfile Lint (Hadolint)') {
      steps {
        sh 'docker run --rm -i hadolint/hadolint hadolint --failure-threshold error - < Dockerfile'
      }
    }
    stage('Build Image') {
      steps { sh 'docker build -t $IMAGE:$TAG .' }
    }
    stage('Image Scan (Trivy)') {
      steps {
        sh '''
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache aquasec/trivy:latest image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 0 $IMAGE:$TAG
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache aquasec/trivy:latest image --severity CRITICAL --ignore-unfixed --exit-code 1 $IMAGE:$TAG
        '''
      }
    }
    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
          usernameVariable: 'U', passwordVariable: 'P')]) {
          sh 'echo $P | docker login -u $U --password-stdin'
          sh 'docker push $IMAGE:$TAG'
        }
      }
    }
    stage('Deploy to K8s') {
      steps {
        sh "sed -i 's/IMAGE_TAG/$TAG/' k8s/deployment.yaml"
        sh 'kubectl apply -f k8s/'
        sh 'kubectl rollout status deployment/healthsite'
      }
    }
  }
}
