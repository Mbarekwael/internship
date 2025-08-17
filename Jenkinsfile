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
    image: node:20-alpine
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
    PROJECT_NAME     = "jenkins "
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
                  set -eux
                  npm ci

                  # Lint only if present; non-blocking
                  if npm run | grep -q "^  lint$"; then
                    echo "Running frontend lint (non-blocking)..."
                    npm run lint || echo "[WARN] Frontend lint failed, continuing..."
                  else
                    echo "[INFO] No 'lint' script in frontend/package.json, skipping."
                  fi

                  # Build only if present
                  if npm run | grep -q "^  build$"; then
                    npm run build
                  else
                    echo "[INFO] No 'build' script in frontend/package.json, skipping."
                  fi
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
                  set -eux
                  npm ci

                  # Lint only if present; non-blocking
                  if npm run | grep -q "^  lint$"; then
                    echo "Running backend lint (non-blocking)..."
                    npm run lint || echo "[WARN] Backend lint failed, continuing..."
                  else
                    echo "[INFO] No 'lint' script in backend/package.json, skipping."
                  fi

                  # Build only if present
                  if npm run | grep -q "^  build$"; then
                    npm run build
                  else
                    echo "[INFO] No 'build' script in backend/package.json, skipping."
                  fi
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
              withCredentials([usernamePassword(credentialsId: 'quay-credentials',
                                               usernameVariable: 'QUAY_USER',
                                               passwordVariable: 'QUAY_PASS')]) {
                sh '''
                  set -eux
                  # Build (rootless-friendly)
                  buildah bud --userns=keep-id --storage-driver=vfs \
                    -t ${FRONTEND_IMAGE} -f frontend/Dockerfile frontend

                  # Push with inline creds (no login => avoids uid_map writes)
                  buildah push --storage-driver=vfs \
                    --creds "$QUAY_USER:$QUAY_PASS" \
                    ${FRONTEND_IMAGE} docker://${FRONTEND_IMAGE}
                '''
              }
            }
          }
        }

        stage('Backend Image') {
          steps {
            container('buildah') {
              withCredentials([usernamePassword(credentialsId: 'quay-credentials',
                                               usernameVariable: 'QUAY_USER',
                                               passwordVariable: 'QUAY_PASS')]) {
                sh '''
                  set -eux
                  # Build (rootless-friendly)
                  buildah bud --userns=keep-id --storage-driver=vfs \
                    -t ${BACKEND_IMAGE} -f backend/Dockerfile backend

                  # Push with inline creds (no login => avoids uid_map writes)
                  buildah push --storage-driver=vfs \
                    --creds "$QUAY_USER:$QUAY_PASS" \
                    ${BACKEND_IMAGE} docker://${BACKEND_IMAGE}
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
              set -eux
              oc login --token=$OC_TOKEN --server=${OPENSHIFT_SERVER} --insecure-skip-tls-verify
              oc project ${PROJECT_NAME}

              if [ -d k8s ]; then
                oc apply -f k8s/
              else
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
