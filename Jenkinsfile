  environment {
    IMAGE_TAG = "${env.BUILD_NUMBER}"
    DOCKER_IMAGE = "hwijin12/frontend:${env.BUILD_NUMBER}"
    DOCKERHUB_CREDENTIALS_ID = "dockerhub-cred"
  }

  stages {
    stage('Clone') {
      steps {
        git branch: 'master', url: 'https://github.com/phwij/msa.git'
      }
    }

    stage('Build') {
      steps {
        container('docker-cli') {
          dir('microservices-demo/src/frontend') {
            sh 'pwd'
            sh 'ls -al'            // ← 여기 오타 있었음: sh'ls -al' → sh 'ls -al'
            sh 'docker build -t $DOCKER_IMAGE .'
          }
        }
      }
    }

    stage('Push') {
      steps {
        container('docker-cli') {
          withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh '''
              set -euo pipefail
              echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
              docker push $DOCKER_IMAGE
            '''
          }
        }
      }
    }
  }  // ← stages 닫기

  post {
    success {
      echo '✅ 사이드카 DinD 기반 빌드 + 배포 성공!'
    }
    failure {
      echo '❌ 실패했으니 로그 확인 바람!'
    }
  }
}  // ← pipeline 닫기

