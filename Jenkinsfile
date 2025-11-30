pipeline {
  agent any

  tools {
    jdk 'jdk'
    maven 'maven'
  }

  environment {
    NEXUS_URL  = 'http://192.168.49.2:30001'
    NEXUS_REPO = 'maven-releases'
    GIT_URL    = 'https://github.com/LeoRC17/hello-world-k8s.git'
    APP_VERSION = ''
    APP_ARTIFACT_ID = ''
    APP_GROUP_ID = ''
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
          // Helper to robustly capture a Maven expression’s last line
          def mvnEval = { expr ->
            sh(
              script: "mvn -q -DforceStdout help:evaluate -Dexpression=${expr} | tail -n 1",
              returnStdout: true
            ).trim()
          }

          try {
            def pom = readMavenPom file: 'pom.xml'
            env.APP_VERSION     = pom.version
            env.APP_ARTIFACT_ID = pom.artifactId
            env.APP_GROUP_ID    = pom.groupId ?: ''
            echo "Resolved from POM -> version=${env.APP_VERSION}, artifactId=${env.APP_ARTIFACT_ID}, groupId=${env.APP_GROUP_ID}"
          } catch (ignored) {
            echo "readMavenPom not available or blocked, falling back to Maven evaluation"
            env.APP_VERSION     = mvnEval('project.version')
            env.APP_ARTIFACT_ID = mvnEval('project.artifactId')
            env.APP_GROUP_ID    = mvnEval('project.groupId')

            echo "Resolved via Maven -> version=${env.APP_VERSION}, artifactId=${env.APP_ARTIFACT_ID}, groupId=${env.APP_GROUP_ID}"
          }

          // Fail fast if we still couldn't resolve metadata
          if (!env.APP_VERSION || !env.APP_ARTIFACT_ID) {
            error "Unable to resolve artifact metadata (version='${env.APP_VERSION}', artifactId='${env.APP_ARTIFACT_ID}'). Check script approval or fallback commands."
          }
        }
      }
    }

    stage('Push to Nexus') {
      steps {
        script {
          def filePath = "target/${env.APP_ARTIFACT_ID}-${env.APP_VERSION}.jar"
          def groupPath = env.APP_GROUP_ID ? env.APP_GROUP_ID.replace('.', '/') : 'com/example'
          def uploadUrl = "${env.NEXUS_URL}/repository/${env.NEXUS_REPO}/${groupPath}/${env.APP_ARTIFACT_ID}/${env.APP_VERSION}/${env.APP_ARTIFACT_ID}-${env.APP_VERSION}.jar"

          withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            // Use shell expansion, not Groovy interpolation, to avoid secret warnings
            sh '''
              set -e
              test -f '"${filePath}"' || (echo "File not found: ${filePath}" && exit 1)
              curl --fail --show-error \
                -u "$NEXUS_USER:$NEXUS_PASS" \
                --upload-file '"${filePath}"' \
                '"${uploadUrl}"'
            '''
          }
        }
      }
    }
  }

  post {
    always { echo 'Pipeline finished' }
  }
}
