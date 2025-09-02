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

    // GitOps 관련
    GITOPS_REPO = "https://github.com/phwij/msa.git'"           // ← 본인 GitOps 저장소로 변경
    GITOPS_TARGET_FILE = "microservices-demo/kubernetes-manifest/frontend.yaml"      // ← 실제 경로 확인
    FRONTEND_CONTAINER_NAME = "server"                             // ← frontend.yaml 안 컨테이너 이름
    GITOPS_CRED_ID = "jenkins"                              // ← Jenkins에 등록된 SSH Key ID

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
            sh 'ls -al'
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

    stage('Update GitOps Repo (frontend only)') {
      steps {
        container('docker-cli') {
          withCredentials([string(credentialsId: 'github-pat', variable: 'GH_TOKEN')]) {
            sh '''
              set -euo pipefail

              # GitOps 리포 주소(https). 토큰 포함된 클론/푸시 URL을 따로 구성
              REPO_HOST="github.com"
              REPO_PATH="phwij/msa-gitops.git"
              CLONE_URL="https://${GH_TOKEN}@${REPO_HOST}/${REPO_PATH}"
  
              # 필요 바이너리
              apk add --no-cache git bash >/dev/null 2>&1 || true


              rm -rf msa-gitops
              git clone "$GITOPS_REPO" msa-gitops
              cd msa-gitops

              TARGET_FILE="${GITOPS_TARGET_FILE:-kubernetes-manifests/frontend.yaml}"
              CNAME="${FRONTEND_CONTAINER_NAME:-server}"

              test -f "$GITOPS_TARGET_FILE" || { echo "❌ Not found: $GITOPS_TARGET_FILE"; exit 1; }

              echo '--- BEFORE ---'
              grep -nE '^[[:space:]]*image:[[:space:]]' "$GITOPS_TARGET_FILE" || true

              # name: server 다음에 나오는 image: 만 NEW_IMAGE로 교체 (다른 이미지 오염 방지)
              awk -v NEW_IMAGE="$DOCKER_IMAGE" -v CNAME="$FRONTEND_CONTAINER_NAME" '
                BEGIN { in_target=0 }
                $0 ~ "name:[[:space:]]*"CNAME"[[:space:]]*$" { in_target=1; print; next }
                in_target==1 && $1 ~ /^image:/ {
                  sub(/^[[:space:]]*image:[[:space:]].*/, "        image: " NEW_IMAGE);
                  in_target=0; print; next
                }
                { print }
              ' "$GITOPS_TARGET_FILE" > "$GITOPS_TARGET_FILE.tmp" && mv "$GITOPS_TARGET_FILE.tmp" "$GITOPS_TARGET_FILE"

              echo '--- AFTER ---'
              grep -nE '^[[:space:]]*image:[[:space:]]' "$GITOPS_TARGET_FILE" || true

              git config user.email "jenkins@ci.local"
              git config user.name  "Jenkins CI"
              git add "$GITOPS_TARGET_FILE"
              git commit -m "Update frontend image to $DOCKER_IMAGE" || echo "No changes to commit"
              git push origin main
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo '✅ DinD 빌드 + DockerHub Push + GitOps(frontend만) 업데이트 완료!'
      echo '🔔 ArgoCD에 Git webhook이 연결돼 있으면 자동으로 롤링됩니다.'
    }
    failure {
      echo '❌ 실패했어요. 각 단계 로그를 확인해주세요.'
    }
  }
}

