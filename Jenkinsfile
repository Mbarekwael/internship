podTemplate(
  containers: [
   
    containerTemplate(
      name: 'jnlp',
      image: 'jenkins/inbound-agent:latest',   
      alwaysPullImage: false
     
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
    hostPathVolume(mountPath: '/run/podman/podman.sock', hostPath: '/run/podman/podman.sock')
  ]
) {
  node(POD_LABEL) {
    def namespace = "beta"
    def imageBackend  = "quay.io/waelmbarek/jobportal-backend:latest"
    def imageFrontend = "quay.io/waelmbarek/jobportal-frontend:latest"

    stage('Clone Repository') {
      git branch: 'main',
          credentialsId: 'github-credentials',
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

    stage('Push Images to Quay') {
      container('podman') {
        withCredentials([usernamePassword(credentialsId: 'quay-credentials', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
          sh """
            podman login quay.io -u \$REG_USER -p \$REG_PASS
            podman push ${imageBackend}
            podman push ${imageFrontend}
          """
        }
      }
    }

    stage('Deploy to OpenShift') {
      container('oc-cli') {
        sh """
          oc project ${namespace} || oc new-project ${namespace}
          oc apply -f k8s/
        """
      }
    }
  }
}
