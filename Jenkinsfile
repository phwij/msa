pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-dind-sidecar
spec:
  volumes:                            #dockerd는 유닉스 소켓을 통해 client와 통신, 해당 소켓을(docker.sock) 바인딩
    - name: docker-sock    
      emptyDir: {}                    #도커데몬(dockerd)이 바인딩 위치를 empydir로 공유 

  containers:
    - name: dind
      image: docker:24.0.7-dind       #dind 컨테이너는 내부의 도커 데몬을 실행
      securityContext:
        privileged: true
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""
      args:
        - dockerd
        - --host=unix:///var/run/docker.sock  #해당 위치에 소켓파일이 생썽
      volumeMounts:
        - name: docker-sock           # 위의 바인딩 위치를 /var/run으로 설정
          mountPath: /var/run
    
    - name: docker-cli
      image: docker:24.0.7
      command: ['sleep', 'infinity']  # CLI 용으로 띄우기 위한 dummy
      volumeMounts:
        - name: docker-sock           # 동일한 /var/run/docker.sock을 공유, cli명령실행
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
        - name: docker-sock          # 동일한 /var/run/docker.sock을 공유, cli명령실행
          mountPath: /var/run        #jnlp를 런타임에 자동주입 / 따라서 local docker socker과 비교했을때 따로 home/jenkins/agent를 마운트 하지 않아도 정상 동작한것
"""
    }
  }

  environment {
    DOCKER_IMAGE = "hwijin12/sidcardocker:${BUILD_NUMBER}"
    DOCKERHUB_CREDENTIALS_ID = "dockerhub-cred"
  }

  stages {
    stage('Clone') {
      steps {
        git branch: 'main', url: 'https://github.com/phwij/sidecardocker.git'
      }
    }

    stage('Build') {
      steps {
        container('docker-cli') {
          dir('apache/exam') {
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

    stage('Deploy') {
      steps {
        container('docker-cli') {
          sh '''
            apk add --no-cache curl bash kubectl > /dev/null
            kubectl set image deployment/apache apache=$DOCKER_IMAGE -n default --record
            kubectl rollout status deployment/apache -n default
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
