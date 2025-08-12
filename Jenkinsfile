pipeline {
    agent any

    environment {
        PROJECT_NAME = "beta"
        OPENSHIFT_SERVER = "https://api.ocp.smartek.ae:6443"
        REGISTRY_CREDENTIALS = 'quay-credentials'  
        FRONTEND_IMAGE = "quay.io/waelmbarek/jobportal-frontend:latest"
        BACKEND_IMAGE = "quay.io/waelmbarek/jobportal-backend:latest"
    }

    stages {
        stage('Clone Repo') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push to Quay with Buildah') {
            parallel {
                stage('Frontend') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", 
                                                           passwordVariable: 'QUAY_PASS', 
                                                           usernameVariable: 'QUAY_USER')]) {
                            sh '''
                                echo "Logging into Quay..."
                                buildah login -u "$QUAY_USER" -p "$QUAY_PASS" quay.io

                                echo "Building frontend image..."
                                cd frontend
                                buildah bud --storage-driver=vfs -t ${FRONTEND_IMAGE} .

                                echo "Pushing frontend image..."
                                buildah push --storage-driver=vfs ${FRONTEND_IMAGE}
                            '''
                        }
                    }
                }

                stage('Backend') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", 
                                                           passwordVariable: 'QUAY_PASS', 
                                                           usernameVariable: 'QUAY_USER')]) {
                            sh '''
                                echo "Logging into Quay..."
                                buildah login -u "$QUAY_USER" -p "$QUAY_PASS" quay.io

                                echo "Building backend image..."
                                cd backend
                                buildah bud --storage-driver=vfs -t ${BACKEND_IMAGE} .

                                echo "Pushing backend image..."
                                buildah push --storage-driver=vfs ${BACKEND_IMAGE}
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploy from Quay to OpenShift') {
            steps {
                withCredentials([string(credentialsId: 'oc-token-id', variable: 'OC_TOKEN')]) {
                    sh '''
                        echo "Deploying to OpenShift..."
                        oc login --token=$OC_TOKEN --server=${OPENSHIFT_SERVER} --insecure-skip-tls-verify
                        oc project ${PROJECT_NAME}

                        oc delete all -l app=jobportal-frontend --ignore-not-found=true
                        oc delete all -l app=jobportal-backend --ignore-not-found=true

                        sleep 10

                        oc new-app ${FRONTEND_IMAGE} --name=jobportal-frontend
                        oc expose svc/jobportal-frontend

                        oc new-app ${BACKEND_IMAGE} --name=jobportal-backend
                        oc expose svc/jobportal-backend

                        oc get pods
                        oc get svc
                        oc get routes
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
        }
    }
}
