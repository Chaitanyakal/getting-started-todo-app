pipeline {
  agent any

  triggers {
    githubPush()
  }

  environment {
    AWS_REGION = 'ap-southeast-2'
    AWS_ACCOUNT_ID = '075810104493'
    IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/todo-app"
    IMAGE_TAG = "${BUILD_NUMBER}"
    EKS_CLUSTER = 'devops-project3'
    NAMESPACE = 'todo'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Test') {
      steps {
        sh '''
          set -e
          docker run --rm \
            -v "$PWD/backend:/workspace" \
            -w /workspace \
            node:22-bookworm \
            sh -c "npm install && npm test"
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker build -t ${IMAGE_URI}:${IMAGE_TAG} .'
      }
    }

    stage('Trivy Scan') {
      steps {
        sh '''
          trivy image \
            --severity HIGH,CRITICAL \
            --exit-code 0 \
            ${IMAGE_URI}:${IMAGE_TAG}
        '''
      }
    }

    stage('Push ECR') {
      steps {
        sh '''
          aws ecr get-login-password --region ${AWS_REGION} |
          docker login --username AWS --password-stdin \
            ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

          docker push ${IMAGE_URI}:${IMAGE_TAG}
        '''
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          aws eks update-kubeconfig \
            --region ${AWS_REGION} \
            --name ${EKS_CLUSTER}

          kubectl apply -f k8s/deployment.yaml

          kubectl set image deployment/todo-app \
            todo-app=${IMAGE_URI}:${IMAGE_TAG} \
            -n ${NAMESPACE}
        '''
      }
    }

    stage('Verify') {
      steps {
        sh '''
          kubectl rollout status deployment/todo-app \
            -n ${NAMESPACE} --timeout=300s

          kubectl get pods -n ${NAMESPACE}
          kubectl get svc -n ${NAMESPACE}
        '''
      }
    }
  }
}
