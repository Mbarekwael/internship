pipeline {
    agent any

    environment {
        PROJECT_NAME = "beta"
        OPENSHIFT_SERVER = "https://api.ocp.smartek.ae:6443"
        REGISTRY_CREDENTIALS = 'quay-credentials'  
        FRONTEND_IMAGE = "quay.io/waelmbarek/jobportal-frontend:p"
        BACKEND_IMAGE = "quay.io/waelmbarek/jobportal-backend:p"
    }

    stages {
        stage('Clone Repo') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push to Quay') {
            parallel {
                stage('Frontend') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", passwordVariable: 'QUAY_PASS', usernameVariable: 'QUAY_USER')]) {
                            sh '''
                                echo "Logging into Quay..."
                                podman login -u "$QUAY_USER" -p "$QUAY_PASS" quay.io

                                echo "Building frontend image..."
                                cd frontend
                                podman build -t ${FRONTEND_IMAGE} .

                                echo "Pushing frontend image..."
                                podman push ${FRONTEND_IMAGE}
                            '''
                        }
                    }
                }

                stage('Backend') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", passwordVariable: 'QUAY_PASS', usernameVariable: 'QUAY_USER')]) {
                            sh '''
                                echo "Logging into Quay..."
                                podman login -u "$QUAY_USER" -p "$QUAY_PASS" quay.io

                                echo "Building backend image..."
                                cd backend
                                podman build -t ${BACKEND_IMAGE} .

                                echo "Pushing backend image..."
                                podman push ${BACKEND_IMAGE}
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploy from Quay') {
            steps {
                withCredentials([string(credentialsId: 'oc-token-id', variable: 'OC_TOKEN')]) {
                    sh '''
                        echo "Deploying from Quay..."
                        oc login --token=$OC_TOKEN --server=${OPENSHIFT_SERVER} --insecure-skip-tls-verify
                        oc project ${PROJECT_NAME}

                        oc delete all -l app=job-frontend --ignore-not-found=true
                        oc delete all -l app=job-backend --ignore-not-found=true

                        sleep 10

                        oc new-app ${FRONTEND_IMAGE} --name=job-frontend
                        oc expose svc/job-frontend

                        oc new-app ${BACKEND_IMAGE} --name=job-backend
                        oc expose svc/job-backend

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
