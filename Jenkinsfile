pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-buildah-node
spec:
  containers:
  - name: nodejs
    image: node:18-alpine
    command: ["cat"]
    tty: true

  - name: buildah
    image: quay.io/buildah/stable:v1.35.4
    command: ["cat"]
    tty: true
    env:
      - name: STORAGE_DRIVER
        value: vfs
      - name: BUILDAH_ISOLATION
        value: chroot
      - name: BUILDAH_USERNS
        value: keep-id
      - name: XDG_RUNTIME_DIR
        value: /tmp/run
      - name: REGISTRY_AUTH_FILE
        value: /home/jenkins/agent/.auth/containers-auth.json
    volumeMounts:
      - name: containers
        mountPath: /var/lib/containers
      - name: xdg
        mountPath: /tmp/run
      - name: authdir
        mountPath: /home/jenkins/agent/.auth

  - name: oc
    image: quay.io/openshift/origin-cli:latest
    command: ["cat"]
    tty: true

  volumes:
  - name: containers
    emptyDir: {}
  - name: xdg
    emptyDir: {}
  - name: authdir
    emptyDir: {}
"""
    }
  }

  environment {
    PROJECT_NAME     = "jenkins"
    OPENSHIFT_SERVER = "https://api.ocp4.smartek.ae:6443"
    FRONTEND_IMAGE   = "quay.io/waelmbarek/jobportal-frontend:latest"
    BACKEND_IMAGE    = "quay.io/waelmbarek/jobportal-backend:latest"
  }

  stages {
    stage('Checkout (SCM)') {
      steps { checkout scm }
    }

    stage('Install & Build') {
      parallel {
        stage('Frontend') {
          steps {
            container('nodejs') {
              dir('frontend') {
                sh '''
                  npm ci
                  npm run lint
                  npm run build
                '''
              }
            }
          }
        }
        stage('Backend') {
          steps {
            container('nodejs') {
              dir('backend') {
                sh '''
                  npm ci
                  npm run lint
                '''
              }
            }
          }
        }
      }
    }

    stage('Build & Push Images (Buildah)') {
      parallel {
        stage('Frontend Image') {
          steps {
            container('buildah') {
              withCredentials([usernamePassword(credentialsId: 'quay-credentials', usernameVariable: 'QUAY_USER', passwordVariable: 'QUAY_PASS')]) {
                sh '''
                  mkdir -p "$(dirname "$REGISTRY_AUTH_FILE")"
                  echo "Login to Quay"
                  buildah login --authfile "$REGISTRY_AUTH_FILE" -u "$QUAY_USER" -p "$QUAY_PASS" quay.io

                  echo "Build frontend image"
                  buildah bud --userns=keep-id --storage-driver=vfs -t ${FRONTEND_IMAGE} -f frontend/Dockerfile frontend

                  echo "Push frontend image"
                  buildah push --authfile "$REGISTRY_AUTH_FILE" --storage-driver=vfs ${FRONTEND_IMAGE} docker://${FRONTEND_IMAGE}
                '''
              }
            }
          }
        }

        stage('Backend Image') {
          steps {
            container('buildah') {
              withCredentials([usernamePassword(credentialsId: 'quay-credentials', usernameVariable: 'QUAY_USER', passwordVariable: 'QUAY_PASS')]) {
                sh '''
                  mkdir -p "$(dirname "$REGISTRY_AUTH_FILE")"
                  echo "Login to Quay"
                  buildah login --authfile "$REGISTRY_AUTH_FILE" -u "$QUAY_USER" -p "$QUAY_PASS" quay.io

                  echo "Build backend image"
                  buildah bud --userns=keep-id --storage-driver=vfs -t ${BACKEND_IMAGE} -f backend/Dockerfile backend

                  echo "Push backend image"
                  buildah push --authfile "$REGISTRY_AUTH_FILE" --storage-driver=vfs ${BACKEND_IMAGE} docker://${BACKEND_IMAGE}
                '''
              }
            }
          }
        }
      }
    }

    stage('Deploy to OpenShift') {
      steps {
        container('oc') {
          withCredentials([string(credentialsId: 'oc-token-id', variable: 'OC_TOKEN')]) {
            sh '''
              oc login --token=$OC_TOKEN --server=${OPENSHIFT_SERVER} --insecure-skip-tls-verify
              oc project ${PROJECT_NAME}

              # If you have manifests in k8s/, apply them:
              if [ -d k8s ]; then
                oc apply -f k8s/
              else
                # Fallback: create apps directly from images
                oc delete all -l app=jobportal-frontend --ignore-not-found
                oc delete all -l app=jobportal-backend --ignore-not-found

                oc new-app ${FRONTEND_IMAGE} --name=jobportal-frontend
                oc expose svc/jobportal-frontend

                oc new-app ${BACKEND_IMAGE} --name=jobportal-backend
                oc expose svc/jobportal-backend
              fi

              oc get pods
              oc get svc
              oc get routes
            '''
          }
        }
      }
    }
  }

  post {
    success { echo '✅ Pipeline completed successfully.' }
    failure { echo '❌ Pipeline failed — check stage logs.' }
    always  { echo 'ℹ️  Pipeline finished.' }
  }
}
