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
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main',
                    credentialsId: "${GIT_CREDENTIALS}",
                    url: 'https://github.com/Mbarekwael/internship.git'
            }
        }

        stage('Build Backend Image') {
            steps {
                script {
                    dir('backend') {
                        sh 'podman build -t ${IMAGE_BACKEND} .'
                    }
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                script {
                    dir('frontend') {
                        sh 'podman build -t ${IMAGE_FRONTEND} .'
                    }
                }
            }
        }

        stage('Push Images to Quay.io') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                    sh "podman login -u $REG_USER -p $REG_PASS ${REGISTRY}"
                    sh "podman push ${IMAGE_BACKEND}"
                    sh "podman push ${IMAGE_FRONTEND}"
                }
            }
        }

        stage('Deploy PVC for MongoDB') {
            steps {
                script {
                    sh '''
                    oc project ${NAMESPACE} || oc new-project ${NAMESPACE}
                    oc apply -f k8s/pvc.yml || echo "PVC already exists"
                    '''
                }
            }
        }

        stage('Deploy MongoDB') {
            steps {
                script {
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
        }

        stage('Deploy Backend') {
            steps {
                script {
                    sh '''
                    oc delete all -l app=jobportal-backend -n ${NAMESPACE} || true
                    oc new-app --name=jobportal-backend \
                    ${IMAGE_BACKEND} \
                    -e MONGO_URL=mongodb://admin:admin123@mongodb:27017/jobportal \
                    -n ${NAMESPACE}
                    '''
                }
            }
        }

        stage('Deploy Frontend') {
            steps {
                script {
                    sh '''
                    oc delete all -l app=jobportal-frontend -n ${NAMESPACE} || true
                    oc new-app --name=jobportal-frontend ${IMAGE_FRONTEND} -n ${NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline executed successfully!'
        }
        failure {
            echo ' Pipeline failed!'
        }
        always {
            echo ' Pipeline finished.'
        }
    }
}
