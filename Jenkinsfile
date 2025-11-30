pipeline {
  agent any

  tools {
    jdk 'jdk'
    maven 'maven'
  }

  environment {
    NEXUS_URL = 'http://192.168.49.2:30001'
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

    stage('Deploy') {
      when { expression { env.ARTIFACT_VERSION && env.ARTIFACT_ID && env.GROUP_ID } }
      steps {
        script {
          // ensure artifact exists
          sh """
            ART=target/${env.ARTIFACT_ID}-${env.ARTIFACT_VERSION}.jar
            echo "Looking for \$ART"
            ls -la target || true
            if [ ! -f "\$ART" ]; then
              echo "ERROR: artifact \$ART not found"
              exit 1
            fi
          """

          withCredentials([usernamePassword(credentialsId: env.NEXUS_CREDENTIALS_ID,
                                            usernameVariable: 'NEXUS_USER',
                                            passwordVariable: 'NEXUS_PASS')]) {
            // create temporary settings.xml and run mvn deploy with verbose output
            // use single-quoted sh block so Groovy does not interpolate secrets; shell expands $NEXUS_USER/$NEXUS_PASS
            sh '''
              cat > ci-settings.xml <<EOF
              <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
                        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                        xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
                <servers>
                  <server>
                    <id>nexus-releases</id>
                    <username>$NEXUS_USER</username>
                    <password>$NEXUS_PASS</password>
                  </server>
                  <server>
                    <id>nexus-snapshots</id>
                    <username>$NEXUS_USER</username>
                    <password>$NEXUS_PASS</password>
                  </server>
                </servers>
              </settings>
              EOF

              # Run deploy with debug so upload lines are visible; tee to capture output
              mvn -B -DskipTests -X deploy --settings ci-settings.xml | tee mvn-deploy.log
              tail -n 200 mvn-deploy.log || true
            '''
          }

          // cleanup
          sh 'rm -f ci-settings.xml mvn-deploy.log || true'
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
