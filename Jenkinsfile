podTemplate(
    containers: [
        containerTemplate(
            name: 'jnlp',
            image: 'jenkins/inbound-agent:latest',
            args: '${computer.jnlpmac} ${computer.name}',
            ttyEnabled: true
        ),
        containerTemplate(
            name: 'podman',
            image: 'quay.io/waelmbarek/podman-agent:v1',
            command: 'cat',
            ttyEnabled: true,
            privileged: true
        ),
        containerTemplate(
            name: 'oc-cli',
            image: 'quay.io/openshift/origin-cli:latest',
            command: 'cat',
            ttyEnabled: true
        )
    ],
    volumes: [
        hostPathVolume(
            mountPath: '/run/podman/podman.sock',
            hostPath: '/run/podman/podman.sock'
        )
    ],
    idleMinutes: 5,
    serviceAccount: 'jenkins', // Ensure your SA has needed permissions
    workspaceVolume: emptyDirWorkspaceVolume(memory: false)
) {
    node(POD_LABEL) {
        def registry = "quay.io"
        def imageFrontend = "quay.io/waelmbarek/jobportal-frontend:latest"
        def imageBackend = "quay.io/waelmbarek/jobportal-backend:latest"
        def gitCredentials = "github-credentials"
        def registryCredentials = "quay-credentials"
        def namespace = "beta"

        stage('Clone Repository') {
            git branch: 'main',
                credentialsId: gitCredentials,
                url: 'https://github.com/Mbarekwael/internship.git'
        }

        stage('Build Backend Image') {
            container('podman') {
                dir('backend') {
                    sh "podman build -t ${imageBackend} ."
                }
            }
        }

        stage('Build Frontend Image') {
            container('podman') {
                dir('frontend') {
                    sh "podman build -t ${imageFrontend} ."
                }
            }
        }

        stage('Push Images to Quay.io') {
            container('podman') {
                withCredentials([usernamePassword(credentialsId: registryCredentials, usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                    sh '''
                    podman login -u $REG_USER -p $REG_PASS quay.io
                    podman push quay.io/waelmbarek/jobportal-backend:latest
                    podman push quay.io/waelmbarek/jobportal-frontend:latest
                    '''
                }
            }
        }

        stage('Deploy to OpenShift') {
            container('oc-cli') {
                sh """
                oc project ${namespace} || oc new-project ${namespace}
                oc apply -f k8s/pvc.yml || echo "PVC already exists"
                oc delete all -l app=mongodb -n ${namespace} || true
                oc new-app --name=mongodb \
                    quay.io/waelmbarek/mongodb \
                    -e MONGO_INITDB_ROOT_USERNAME=admin \
                    -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
                    --volume=beta-db:/data/db \
                    -n ${namespace} || echo "MongoDB deployment skipped"
                oc delete all -l app=jobportal-backend -n ${namespace} || true
                oc new-app --name=jobportal-backend \
                    ${imageBackend} \
                    -e MONGO_URL=mongodb://admin:admin123@mongodb:27017/jobportal \
                    -n ${namespace}
                oc delete all -l app=jobportal-frontend -n ${namespace} || true
                oc new-app --name=jobportal-frontend ${imageFrontend} -n ${namespace}
                """
            }
        }

        post {
            success { echo 'CI/CD pipeline executed successfully!' }
            failure { echo 'Pipeline failed!' }
            always { echo 'Pipeline finished.' }
        }
    }
}
