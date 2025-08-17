pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: quay.io/sisense/jenkins/inbound-agent:3327.v868139a_d00e0-2
  - name: buildah
    image: quay.io/buildah/stable:v1.35.4
    command: ["cat"]
    tty: true
    env:
      - name: STORAGE_DRIVER
        value: vfs
      - name: BUILDAH_ISOLATION
        value: chroot
      - name: XDG_RUNTIME_DIR
        value: /tmp/run
    volumeMounts:
      - name: containers
        mountPath: /var/lib/containers
      - name: xdg
        mountPath: /tmp/run
  volumes:
    - name: containers
      emptyDir: {}
    - name: xdg
      emptyDir: {}
"""
    }
  }

  environment {
    PROJECT_NAME = "beta"
    OPENSHIFT_SERVER = "https://api.ocp4.smartek.ae"
    REGISTRY_CREDENTIALS = 'quay-credentials'
    FRONTEND_IMAGE = "quay.io/waelmbarek/jobportal-frontend:latest"
    BACKEND_IMAGE  = "quay.io/waelmbarek/jobportal-backend:latest"
  }

  stages {
    stage('Clone Repo') {
      steps { checkout scm }
    }

    stage('Build & Push to Quay (Buildah)') {
      parallel {
        stage('Frontend') {
          steps {
            container('buildah') {
              withCredentials([usernamePassword(credentialsId: REGISTRY_CREDENTIALS, usernameVariable: 'QUAY_USER', passwordVariable: 'QUAY_PASS')]) {
                sh '''
                  buildah login -u "$QUAY_USER" -p "$QUAY_PASS" quay.io
                  cd frontend
                  buildah bud --storage-driver=vfs -t ${FRONTEND_IMAGE} .
                  buildah push --storage-driver=vfs ${FRONTEND_IMAGE}
                '''
              }
            }
          }
        }
        stage('Backend') {
          steps {
            container('buildah') {
              withCredentials([usernamePassword(credentialsId: REGISTRY_CREDENTIALS, usernameVariable: 'QUAY_USER', passwordVariable: 'QUAY_PASS')]) {
                sh '''
                  buildah login -u "$QUAY_USER" -p "$QUAY_PASS" quay.io
                  cd backend
                  buildah bud --storage-driver=vfs -t ${BACKEND_IMAGE} .
                  buildah push --storage-driver=vfs ${BACKEND_IMAGE}
                '''
              }
            }
          }
        }
      }
    }

    stage('Deploy to OpenShift') {
      steps {
        container('buildah') {
          withCredentials([string(credentialsId: 'oc-token-id', variable: 'OC_TOKEN')]) {
            sh '''
              oc login --token=$OC_TOKEN --server=${OPENSHIFT_SERVER} --insecure-skip-tls-verify
              oc project ${PROJECT_NAME}
              oc apply -f k8s/
            '''
          }
        }
      }
    }
  }
}
