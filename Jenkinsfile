pipeline {
  agent any

  tools {
    jdk 'jdk'
    maven 'maven'
  }

  environment {
    // Keep NEXUS_URL as the base URL (no trailing repository path)
    NEXUS_URL = 'http://192.168.49.2:30001'  
    NEXUS_REPOSITORY = 'repository/maven-releases'   // path segment under the base URL
    NEXUS_CREDENTIALS_ID = 'nexus-creds'             // Jenkins credentials ID and repositoryId
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Verify tools') {
      steps {
        script {
          sh '''
            echo "NODE: $(hostname)"
            echo "JENKINS_HOME: ${JENKINS_HOME}"
            java -version
            mvn -v
          '''
        }
      }
    }

    stage('Build') {
      steps {
        script { sh 'mvn -B clean package -DskipTests' }
      }
      post {
        success {
          echo 'Build successful'
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }
    }

    stage('Resolve artifact metadata') {
      steps {
        script {
          // Use Maven to reliably evaluate project coordinates
          def groupId = sh(script: "mvn help:evaluate -Dexpression=project.groupId -q -DforceStdout", returnStdout: true).trim()
          def artifactId = sh(script: "mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout", returnStdout: true).trim()
          def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
          env.GROUP_ID = groupId
          env.ARTIFACT_ID = artifactId
          env.ARTIFACT_VERSION = version

          echo "Resolved artifact metadata:"
          echo "  GroupId: ${env.GROUP_ID}"
          echo "  ArtifactId: ${env.ARTIFACT_ID}"
          echo "  Version: ${env.ARTIFACT_VERSION}"
        }
      }
    }

    stage('Push to Nexus') {
      when {
        expression { env.ARTIFACT_VERSION && env.ARTIFACT_ID && env.GROUP_ID }
      }
      steps {
        script {
          echo "Pushing ${env.GROUP_ID}:${env.ARTIFACT_ID}:${env.ARTIFACT_VERSION} to Nexus"

          // compute artifact filename and check it exists
          def artifactFile = "target/${env.ARTIFACT_ID}-${env.ARTIFACT_VERSION}.jar"
          sh """
            echo "Looking for artifact: ${artifactFile}"
            ls -la target || true
            if [ ! -f "${artifactFile}" ]; then
              echo "ERROR: Artifact ${artifactFile} not found. Aborting deploy."
              exit 1
            fi
          """

          withCredentials([usernamePassword(
            credentialsId: env.NEXUS_CREDENTIALS_ID,
            usernameVariable: 'NEXUS_USERNAME',
            passwordVariable: 'NEXUS_PASSWORD'
          )]) {
            // Use repositoryId equal to the credentials id so settings.xml server id can match
            sh """
              mvn -B deploy:deploy-file \
                -DgroupId=${env.GROUP_ID} \
                -DartifactId=${env.ARTIFACT_ID} \
                -Dversion=${env.ARTIFACT_VERSION} \
                -Dpackaging=jar \
                -Dfile=${artifactFile} \
                -DrepositoryId=${NEXUS_CREDENTIALS_ID} \
                -Durl=${NEXUS_URL}/${NEXUS_REPOSITORY} \
                -DgeneratePom=true
            """
          }
        }
      }
    }
  }

  post {
    always {
      echo 'Pipeline finished'
      sh 'rm -f version.txt artifactId.txt groupId.txt || true'
    }
    success {
      echo 'Pipeline completed successfully! Artifact deployed to Nexus.'
    }
    failure {
      echo 'Pipeline failed!'
    }
  }
}
