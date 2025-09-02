pipeline {
  triggers {
    githubPush()
  }
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-dind-sidecar
spec:
  volumes:
    - name: docker-sock
      emptyDir: {}  # 도커 데몬이 소켓을 여기다 씀

  containers:
    - name: dind
      image: docker:24.0.7-dind
      securityContext:
        privileged: true
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""
      args:
        - dockerd
        - --host=unix:///var/run/docker.sock
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run
    
    - name: docker-cli
      image: docker:24.0.7
      command: ['sleep', 'infinity']  # CLI 용으로 띄우기 위한 dummy
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run

    - name: jnlp
      image: jenkins/inbound-agent:alpine
      resources:
        limits:
          memory: "512Mi"
          cpu: "500m"
        requests:
          memory: "256Mi"
          cpu: "100m"
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run
"""
    }
  }

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
            sh'ls -al'
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
              echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
              docker push $DOCKER_IMAGE
            '''
          }
        }
      }
    }

  post {
    success {
      echo '✅ 사이드카 DinD 기반 빌드 + 배포 성공!'
    }
    failure {
      echo '❌ 실패했으니 로그 확인 바람!'
    }
  }
}

