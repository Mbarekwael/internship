pipeline {
    agent any

    environment {
        REGISTRY = "quay.io"
        IMAGE_FRONTEND = "quay.io/waelmbarek/jobportal-frontend:latest"
        IMAGE_BACKEND = "quay.io/waelmbarek/jobportal-backend:latest"
        IMAGE_MONGO = "quay.io/waelmbarek/mongodb"
        GIT_CREDENTIALS = "github-credentials"
        REGISTRY_CREDENTIALS = "quay-credentials"
        NAMESPACE = "beta"
        PODMAN_HOME = "${HOME}/.local/podman"
        PATH = "${HOME}/.local/podman/bin:$PATH"
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main',
                    credentialsId: "${GIT_CREDENTIALS}",
                    url: 'https://github.com/Mbarekwael/internship.git'
            }
        }

        stage('Install Podman (Rootless Persistent)') {
            steps {
                script {
                    sh '''
                    set -e

                    echo "[INFO] Setting up persistent Podman rootless install"

                    export PODMAN_HOME=$HOME/.local/podman
                    export PATH=$PODMAN_HOME/bin:$PATH

                    if [ -x "$PODMAN_HOME/bin/podman" ]; then
                      echo "[INFO] Podman already installed at $PODMAN_HOME"
                      $PODMAN_HOME/bin/podman --version
                      exit 0
                    fi

                    mkdir -p $PODMAN_HOME/bin
                    cd $PODMAN_HOME/bin

                    curl -LO https://github.com/containers/podman/releases/download/v4.9.0/podman-remote-static-linux_amd64.tar.gz
                    tar -xvzf podman-remote-static-linux_amd64.tar.gz
                    rm podman-remote-static.tar.gz

                    echo "[INFO] Podman installed to $PODMAN_HOME"
                    ./podman --version
                    '''
                }
            }
        }

        stage('Build Backend Image') {
            steps {
                dir('backend') {
                    sh 'podman build -t ${IMAGE_BACKEND} .'  // This will likely fail using podman-remote!
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                dir('frontend') {
                    sh 'podman build -t ${IMAGE_FRONTEND} .'  // This will likely fail using podman-remote!
                }
            }
        }

        stage('Push Images to Quay.io') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                    sh '''
                    export PATH=$HOME/.local/podman/bin:$PATH
                    podman login -u $REG_USER -p $REG_PASS ${REGISTRY}
                    podman push ${IMAGE_BACKEND}
                    podman push ${IMAGE_FRONTEND}
                    '''
                }
            }
        }

        stage('Deploy PVC for MongoDB') {
            steps {
                sh '''
                oc project ${NAMESPACE} || oc new-project ${NAMESPACE}
                oc apply -f k8s/pvc.yml || echo "PVC already exists"
                '''
            }
        }

        stage('Deploy MongoDB') {
            steps {
                sh '''
                oc delete all -l app=mongodb -n ${NAMESPACE} || true
                oc new-app --name=mongodb \
                ${IMAGE_MONGO} \
                -e MONGO_INITDB_ROOT_USERNAME=admin \
                -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
                --volume=beta-db:/data/db \
                -n ${NAMESPACE} || echo "MongoDB deployment skipped"
                '''
            }
        }

        stage('Deploy Backend') {
            steps {
                sh '''
                oc delete all -l app=jobportal-backend -n ${NAMESPACE} || true
                oc new-app --name=jobportal-backend \
                ${IMAGE_BACKEND} \
                -e MONGO_URL=mongodb://admin:admin123@mongodb:27017/jobportal \
                -n ${NAMESPACE}
                '''
            }
        }

        stage('Deploy Frontend') {
            steps {
                sh '''
                oc delete all -l app=jobportal-frontend -n ${NAMESPACE} || true
                oc new-app --name=jobportal-frontend ${IMAGE_FRONTEND} -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            echo 'Pipeline finished.'
        }
    }
}
