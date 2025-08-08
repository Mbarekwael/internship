podTemplate(
  containers: [
    containerTemplate(
      name: 'podman',
      image: 'vniks/podman-agent:v1',
      command: 'cat',
      privileged: true,
      user: 'root',
      ttyEnabled: true
    ),
    containerTemplate(
      name: 'kubectl',
      image: 'lachlanevenson/k8s-kubectl:latest',
      command: 'cat',
      ttyEnabled: true
    )
  ],
  volumes: [
    hostPathVolume(mountPath: '/run/podman/podman.sock', hostPath: '/run/podman/podman.sock')
]) {

  node(POD_LABEL) {

    stage('Clone Repository') {
      git branch: 'main',
          credentialsId: "github-credentials",
          url: 'https://github.com/Mbarekwael/internship.git'
    }

    container('podman') {
      stage('Build Backend Image') {
        dir('backend') {
          sh 'podman build -t quay.io/waelmbarek/jobportal-backend:latest .'
        }
      }

      stage('Build Frontend Image') {
        dir('frontend') {
          sh 'podman build -t quay.io/waelmbarek/jobportal-frontend:latest .'
        }
      }

      stage('Push Images to Quay.io') {
        withCredentials([usernamePassword(credentialsId: "quay-credentials", usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
          sh '''
            podman login -u $REG_USER -p $REG_PASS quay.io
            podman push quay.io/waelmbarek/jobportal-backend:latest
            podman push quay.io/waelmbarek/jobportal-frontend:latest
          '''
        }
      }
    }

    stage('Deploy to OpenShift') {
      container('kubectl') {
        sh '''
          oc project beta || oc new-project beta
          oc apply -f k8s/pvc.yml || echo "PVC already exists"

          oc delete all -l app=mongodb -n beta || true
          oc new-app --name=mongodb quay.io/waelmbarek/mongodb \
            -e MONGO_INITDB_ROOT_USERNAME=admin \
            -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
            --volume=beta-db:/data/db \
            -n beta || echo "MongoDB deployment skipped"

          oc delete all -l app=jobportal-backend -n beta || true
          oc new-app --name=jobportal-backend quay.io/waelmbarek/jobportal-backend:latest \
            -e MONGO_URL=mongodb://admin:admin123@mongodb:27017/jobportal \
            -n beta

          oc delete all -l app=jobportal-frontend -n beta || true
          oc new-app --name=jobportal-frontend quay.io/waelmbarek/jobportal-frontend:latest -n beta
        '''
      }
    }
  }
}
