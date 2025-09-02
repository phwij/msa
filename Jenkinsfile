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
      emptyDir: {}

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
      command: ['sleep', 'infinity']
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

    // ArgoCD가 보고 있는 msa.git에 바로 커밋/푸시
    REPO_HOST = "github.com"
    REPO_PATH = "phwij/msa.git"
    GIT_BRANCH = "master" // ← ArgoCD targetRevision과 맞추세요

    // 수정할 매니페스트(ArgoCD path)
    GITOPS_TARGET_FILE = "microservices-demo/kubernetes-manifests/frontend.yaml"
    FRONTEND_CONTAINER_NAME = "server"
  }

  stages {
    stage('Clone') {
      steps {
        // 소스 빌드용 체크아웃 (브랜치도 master으로 통일)
        git branch: env.GIT_BRANCH, url: "https://${REPO_HOST}/${REPO_PATH}"
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
              docker push "$DOCKER_IMAGE"
            '''
          }
        }
      }
    }

    stage('Update Repo (frontend only)') {
      steps {
        container('docker-cli') {
          withCredentials([string(credentialsId: 'github-pat', variable: 'GH_TOKEN')]) {
            sh '''
              set -euo pipefail

              # public이라 clone은 토큰 없이, push만 PAT 사용
              CLONE_URL_RO="https://${REPO_HOST}/${REPO_PATH}"
              PUSH_URL="https://${GH_TOKEN}@${REPO_HOST}/${REPO_PATH}"

              # docker:alpine 계열엔 git/bash 없을 수 있음
              apk add --no-cache git bash >/dev/null 2>&1 || true

              rm -rf repo
              git clone "$CLONE_URL_RO" repo
              cd repo
              git checkout "$GIT_BRANCH"

              TARGET_FILE="$GITOPS_TARGET_FILE"
              CNAME="$FRONTEND_CONTAINER_NAME"

              test -f "$TARGET_FILE" || { echo "❌ Not found: $TARGET_FILE"; exit 1; }

              echo '--- BEFORE ---'
              grep -nE "^[[:space:]]*image:[[:space:]]" "$TARGET_FILE" || true

              # name: $CNAME 다음의 image: 라인만 안전 교체
              awk -v NEW_IMAGE="$DOCKER_IMAGE" -v CNAME="$CNAME" '
                BEGIN { in_target=0 }
                $0 ~ "name:[[:space:]]*"CNAME"[[:space:]]*$" { in_target=1; print; next }
                in_target==1 && $1 ~ /^image:/ {
                  sub(/^[[:space:]]*image:[[:space:]].*/, "        image: " NEW_IMAGE);
                  in_target=0; print; next
                }
                { print }
              ' "$TARGET_FILE" > "$TARGET_FILE.tmp" && mv "$TARGET_FILE.tmp" "$TARGET_FILE"

              echo '--- AFTER ---'
              grep -nE "^[[:space:]]*image:[[:space:]]" "$TARGET_FILE" || true

              git config user.email "jenkins@ci.local"
              git config user.name  "Jenkins CI"
              git add "$TARGET_FILE"
              git commit -m "Update frontend image to $DOCKER_IMAGE" || { echo "No changes to commit"; exit 0; }

              git push "$PUSH_URL" HEAD:"$GIT_BRANCH"
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo '✅ DinD 빌드 + DockerHub Push + msa.git(frontend만) 업데이트 완료!'
      echo '🔔 ArgoCD가 repo=msa.git, path=microservices-demo/kubernetes-manifests, rev=master 으로 감지합니다.'
    }
    failure {
      echo '❌ 실패했어요. 각 단계 로그를 확인해주세요.'
    }
  }
}

