pipeline {
  agent any

  environment {
    NEXUS_URL = 'http://192.168.49.2:30001'
    NEXUS_REPO = 'maven-releases'
    GIT_URL = 'https://github.com/LeoRC17/hello-world-k8s.git'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: "${env.GIT_URL}"
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B clean package -DskipTests'
      }
    }

    stage('Push to Nexus') {
      steps {
        script {
          def pom = readMavenPom file: 'pom.xml'
          def version = pom.version
          def artifactId = pom.artifactId
          def filePath = "target/${artifactId}-${version}.jar"

          withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            // Quick upload to the releases repo (works for testing)
            sh """
              curl -u ${NEXUS_USER}:${NEXUS_PASS} --fail --show-error \
                --upload-file ${filePath} \
                ${NEXUS_URL}/repository/${NEXUS_REPO}/com/example/${artifactId}/${version}/${artifactId}-${version}.jar
            """
          }
        }
      }
    }
  }

  post {
    always { echo 'Pipeline finished' }
  }
}
