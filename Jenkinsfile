pipeline {
  agent any
  environment {
    IMAGE       = 'alanjose2929/mediplus'
    REPO        = 'github.com/AlanJose2712/mediplus.git'
    SEMGREP     = '/home/ubuntu/scanvenv/bin/semgrep'
    TRIVY_CACHE = '/home/ubuntu/trivy-cache'
  }
  stages {
    stage('1. Checkout') {
      steps {
        checkout scm
        script {
          def author = sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim()
          env.SKIP = (author == 'jenkins-bot') ? 'true' : 'false'
          echo "Last commit author: ${author} -> skip=${env.SKIP}"
        }
      }
    }
    stage('2. Semgrep SAST') {
      when { expression { env.SKIP != 'true' } }
      steps { sh '${SEMGREP} --config auto --exclude static --exclude media . || true' }
    }
    stage('3. Trivy FS') {
      when { expression { env.SKIP != 'true' } }
      steps { sh 'trivy fs --cache-dir ${TRIVY_CACHE} --severity HIGH,CRITICAL --exit-code 0 .' }
    }
    stage('4. Docker Build') {
      when { expression { env.SKIP != 'true' } }
      steps { sh 'docker build -t ${IMAGE}:${BUILD_NUMBER} -t ${IMAGE}:latest .' }
    }
    stage('5. Trivy Image') {
      when { expression { env.SKIP != 'true' } }
      steps { sh 'trivy image --cache-dir ${TRIVY_CACHE} --severity HIGH,CRITICAL --exit-code 0 ${IMAGE}:${BUILD_NUMBER}' }
    }
    stage('6. Push to Docker Hub') {
      when { expression { env.SKIP != 'true' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DU', passwordVariable: 'DP')]) {
          sh '''
            echo "$DP" | docker login -u "$DU" --password-stdin
            docker push ${IMAGE}:${BUILD_NUMBER}
            docker push ${IMAGE}:latest
          '''
        }
      }
    }
    stage('7. ZAP Scan') {
      when { expression { env.SKIP != 'true' } }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          sh '''
            docker rm -f mediplus-test || true
            docker run -d --name mediplus-test -p 18082:8000 ${IMAGE}:${BUILD_NUMBER}
            sleep 15
            mkdir -p /home/ubuntu/zap-out && chmod 777 /home/ubuntu/zap-out
            docker run --rm --network host -v /home/ubuntu/zap-out:/zap/wrk:rw \
              ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
              -t http://127.0.0.1:18082 -r mediplus-zap.html
          '''
        }
        sh 'docker rm -f mediplus-test || true'
      }
    }
    stage('8. Update Manifest (GitOps)') {
      when { expression { env.SKIP != 'true' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GU', passwordVariable: 'GP')]) {
          sh '''
            git config user.name  "jenkins-bot"
            git config user.email "jenkins-bot@users.noreply.github.com"
            sed -i "s|image: ${IMAGE}:.*|image: ${IMAGE}:${BUILD_NUMBER}|" k8s/deployment.yaml
            git add k8s/deployment.yaml
            git commit -m "Deploy ${BUILD_NUMBER}"
            git push https://${GU}:${GP}@${REPO} HEAD:main
          '''
        }
      }
    }
  }
  post { always { sh 'docker image prune -f || true' } }
}
