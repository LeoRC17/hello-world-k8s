pipeline {
  agent any

  tools {
    jdk 'jdk'
    maven 'maven'
  }

  environment {
    // Base Nexus URL (optional; distributionManagement in pom.xml will determine final repo)
    NEXUS_URL = 'http://192.168.49.2:30001'
    // Jenkins credentials ID that contains Nexus username/password
    NEXUS_CREDENTIALS_ID = 'nexus-creds'
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
          env.GROUP_ID = sh(script: "mvn help:evaluate -Dexpression=project.groupId -q -DforceStdout", returnStdout: true).trim()
          env.ARTIFACT_ID = sh(script: "mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout", returnStdout: true).trim()
          env.ARTIFACT_VERSION = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()

          echo "Resolved artifact metadata:"
          echo "  GroupId: ${env.GROUP_ID}"
          echo "  ArtifactId: ${env.ARTIFACT_ID}"
          echo "  Version: ${env.ARTIFACT_VERSION}"
        }
      }
    }

    stage('Deploy to Nexus') {
      when {
        expression { env.ARTIFACT_VERSION && env.ARTIFACT_ID && env.GROUP_ID }
      }
      steps {
        script {
          // Determine whether this is a snapshot build (pom's distributionManagement will route correctly)
          def isSnapshot = env.ARTIFACT_VERSION?.endsWith('-SNAPSHOT')
          echo "Deploying ${env.GROUP_ID}:${env.ARTIFACT_ID}:${env.ARTIFACT_VERSION} (snapshot=${isSnapshot})"

          // Ensure artifact exists
          sh """
            ARTIFACT=target/${env.ARTIFACT_ID}-${env.ARTIFACT_VERSION}.jar
            echo "Looking for artifact: \$ARTIFACT"
            ls -la target || true
            if [ ! -f "\$ARTIFACT" ]; then
              echo "ERROR: Artifact \$ARTIFACT not found. Aborting deploy."
              exit 1
            fi
          """

          // Create a temporary settings.xml with credentials and run mvn deploy (uses distributionManagement in pom.xml)
          withCredentials([usernamePassword(credentialsId: env.NEXUS_CREDENTIALS_ID,
                                            usernameVariable: 'NEXUS_USER',
                                            passwordVariable: 'NEXUS_PASS')]) {
            sh '''
              cat > ci-settings.xml <<EOF
              <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
                        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                        xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                                            https://maven.apache.org/xsd/settings-1.0.0.xsd">
                <servers>
                  <!-- IDs must match the <id> values in distributionManagement of pom.xml -->
                  <server>
                    <id>nexus-releases</id>
                    <username>${NEXUS_USER}</username>
                    <password>${NEXUS_PASS}</password>
                  </server>
                  <server>
                    <id>nexus-snapshots</id>
                    <username>${NEXUS_USER}</username>
                    <password>${NEXUS_PASS}</password>
                  </server>
                </servers>
              </settings>
              EOF

              # Run Maven deploy which will use distributionManagement in the POM to choose the correct repo
              mvn -B -DskipTests deploy --settings ci-settings.xml
            '''
          }

          // Cleanup temporary settings
          sh 'rm -f ci-settings.xml || true'
        }
      }
    }
  }

  post {
    always {
      echo 'Pipeline finished'
    }
    success {
      echo 'Pipeline completed successfully! Artifact deployed to Nexus.'
    }
    failure {
      echo 'Pipeline failed!'
    }
  }
}
