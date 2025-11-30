pipeline {
  agent any

  tools {
    jdk 'jdk'        // matches the JDK name you configured in Global Tool Configuration
    maven 'maven'    // matches the Maven name you configured in Global Tool Configuration
  }

  environment {
    NEXUS_URL = 'http://192.168.49.2:30001'
    NEXUS_REPO = 'maven-releases'
    GIT_URL = 'https://github.com/LeoRC17/hello-world-k8s.git'
    APP_VERSION = ''
    APP_ARTIFACT_ID = ''
    APP_GROUP_ID = ''   // optional: set from pom if you want to mirror groupId in the Nexus path
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: "${env.GIT_URL}"
      }
    }

    stage('Verify tools') {
      steps {
        sh 'echo NODE: $(hostname)'
        sh 'echo JENKINS_HOME: $JENKINS_HOME'
        sh 'java -version || true'
        sh 'mvn -v || true'
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

    stage('Resolve artifact metadata') {
      steps {
        script {
          try {
            // Preferred: requires Pipeline Utility Steps plugin
            def pom = readMavenPom file: 'pom.xml'
            env.APP_VERSION    = pom.version
            env.APP_ARTIFACT_ID = pom.artifactId
            env.APP_GROUP_ID   = pom.groupId ?: ''
            echo "Resolved from POM -> version=${env.APP_VERSION}, artifactId=${env.APP_ARTIFACT_ID}, groupId=${env.APP_GROUP_ID}"
          } catch (e) {
            // Fallback: use Maven help:evaluate without extra plugins
            echo "readMavenPom not available, falling back to Maven evaluation"
            env.APP_VERSION = sh(
              script: 'mvn -q -DforceStdout help:evaluate -Dexpression=project.version',
              returnStdout: true
            ).trim()
            env.APP_ARTIFACT_ID = sh(
              script: 'mvn -q -DforceStdout help:evaluate -Dexpression=project.artifactId',
              returnStdout: true
            ).trim()
            env.APP_GROUP_ID = sh(
              script: 'mvn -q -DforceStdout help:evaluate -Dexpression=project.groupId',
              returnStdout: true
            ).trim()
            echo "Resolved via Maven -> version=${env.APP_VERSION}, artifactId=${env.APP_ARTIFACT_ID}, groupId=${env.APP_GROUP_ID}"
          }
        }
      }
    }

    stage('Push to Nexus') {
      steps {
        script {
          def filePath = "target/${env.APP_ARTIFACT_ID}-${env.APP_VERSION}.jar"

          // If you want to mirror the Maven coordinates in Nexus, derive the path from groupId
          def groupPath = env.APP_GROUP_ID ? env.APP_GROUP_ID.replace('.', '/') : 'com/example'
          def uploadUrl = "${env.NEXUS_URL}/repository/${env.NEXUS_REPO}/${groupPath}/${env.APP_ARTIFACT_ID}/${env.APP_VERSION}/${env.APP_ARTIFACT_ID}-${env.APP_VERSION}.jar"

          withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            sh """
              test -f ${filePath}
              curl -u ${NEXUS_USER}:${NEXUS_PASS} --fail --show-error \
                --upload-file ${filePath} \
                ${uploadUrl}
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
