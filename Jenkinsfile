pipeline {
  agent any

  environment {
    NEXUS_URL = 'http://nexus-service:8081'   // service name inside k8s
    NEXUS_REPO = 'my-maven-repo'              // change to your Nexus repo name
    GIT_URL = 'https://github.com/LeoRC17/hello-world-k8s.git'
  }

  parameters {
    string(name: 'PUSH_METHOD', defaultValue: 'curl', description: 'curl or maven')
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
      post {
        success { echo 'Build successful' }
        failure { echo 'Build failed' }
      }
    }

    stage('Test') {
      steps {
        sh 'mvn -B test'
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
            if (params.PUSH_METHOD == 'maven') {
              // Maven deploy path (recommended for Maven repos)
              writeFile file: 'ci-settings.xml', text: """
                <settings>
                  <servers>
                    <server>
                      <id>nexus</id>
                      <username>${NEXUS_USER}</username>
                      <password>${NEXUS_PASS}</password>
                    </server>
                  </servers>
                </settings>
              """
              sh "mvn -s ci-settings.xml -B deploy -DskipTests"
            } else {
              // Quick curl upload to a raw/hosted repo path
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

    stage('Build Docker Image') {
      steps {
        sh 'docker build -t hello-world:latest .'
      }
    }
  }

  post {
    always { echo 'Pipeline completed' }
  }
}
