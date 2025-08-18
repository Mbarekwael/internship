pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-nodejs
spec:
  containers:
  - name: nodejs
    image: node:18-alpine
    command: [ "cat" ]
    tty: true
  - name: buildah
    image: quay.io/buildah/stable:latest
    securityContext: { privileged: true }
    command: [ "cat" ]
    tty: true
  - name: oc
    image: quay.io/openshift/origin-cli:latest
    command: [ "cat" ]
    tty: true
"""
    }
  }

  environment {
    OCP_API_URL   = 'https://api.ocp4.smartek.ae:6443'
    OCP_NAMESPACE = 'beta'   

    FRONTEND_IMAGE = 'quay.io/waelmbarek/jobportal-frontend:latest'
    BACKEND_IMAGE  = 'quay.io/waelmbarek/jobportal-backend:latest'

    FRONTEND_BUILD_FILE = 'Containerfile'
    BACKEND_BUILD_FILE  = 'Containerfile'
  }

  stages {
    stage('Clone Repo') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-token-wael',
                                          usernameVariable: 'GH_USER',
                                          passwordVariable: 'GH_TOKEN')]) {
          sh '''
            rm -rf source
            git clone https://${GH_USER}:${GH_TOKEN}@github.com/Mbarekwael/internship.git source
            cd source && git rev-parse --short HEAD
          '''
        }
      }
    }

    stage('Install & Build') {
      parallel {
        stage('Frontend') {
          steps {
            container('nodejs') {
              dir('source/frontend') {
                sh '''
                  npm install
                  npm run build
                '''
              }
            }
          }
        }
        stage('Backend') {
          steps {
            container('nodejs') {
              dir('source/backend') {
                sh '''
                  npm install
                '''
              }
            }
          }
        }
      }
    }

    stage('Build and Push Images') {
      parallel {
        stage('Frontend') {
          steps {
            container('buildah') {
              withCredentials([usernamePassword(credentialsId: 'quay-credentials-wael',
                                                usernameVariable: 'QUAY_USER',
                                                passwordVariable: 'QUAY_PASS')]) {
                sh '''
                  cd source/frontend
                  buildah bud --isolation=chroot -t "$FRONTEND_IMAGE" -f "$FRONTEND_BUILD_FILE" .
                  buildah login -u "$QUAY_USER" -p "$QUAY_PASS" quay.io
                  buildah push "$FRONTEND_IMAGE"
                '''
              }
            }
          }
        }
        stage('Backend') {
          steps {
            container('buildah') {
              withCredentials([usernamePassword(credentialsId: 'quay-credentials-wael',
                                                usernameVariable: 'QUAY_USER',
                                                passwordVariable: 'QUAY_PASS')]) {
                sh '''
                  cd source/backend
                  buildah bud --isolation=chroot -t "$BACKEND_IMAGE" -f "$BACKEND_BUILD_FILE" .
                  buildah login -u "$QUAY_USER" -p "$QUAY_PASS" quay.io
                  buildah push "$BACKEND_IMAGE"
                '''
              }
            }
          }
        }
      }
    }

    stage('Deploy') {
      steps {
        container('oc') {
          withCredentials([string(credentialsId: 'ocp-token-wael', variable: 'OCP_TOKEN')]) {
            sh '''
              oc login --token="$OCP_TOKEN" --server="$OCP_API_URL" --insecure-skip-tls-verify
              oc project "$OCP_NAMESPACE" || oc new-project "$OCP_NAMESPACE"

              if ! oc get deploy/backend-app >/dev/null 2>&1; then
                oc new-app "$BACKEND_IMAGE" --name=backend-app
              else
                oc set image deploy/backend-app backend-app="$BACKEND_IMAGE"
              fi

              if ! oc get deploy/frontend-app >/dev/null 2>&1; then
                oc new-app "$FRONTEND_IMAGE" --name=frontend-app
              else
                oc set image deploy/frontend-app frontend-app="$FRONTEND_IMAGE"
              fi
            '''
          }
        }
      }
    }
  }

  post {
    success { echo ' Built & pushed to Quay' }
    failure { echo 'Pipeline failed. Check logs.' }
  }
}