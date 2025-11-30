stage('Deploy') {
  when { expression { env.ARTIFACT_VERSION && env.ARTIFACT_ID && env.GROUP_ID } }
  steps {
    script {
      // ensure artifact exists
      sh '''
        ART=target/${ARTIFACT_ID}-${ARTIFACT_VERSION}.jar
        echo "Looking for $ART"
        ls -la target || true
        if [ ! -f "$ART" ]; then
          echo "ERROR: artifact $ART not found"
          exit 1
        fi
      '''

      withCredentials([usernamePassword(credentialsId: env.NEXUS_CREDENTIALS_ID,
                                        usernameVariable: 'NEXUS_USER',
                                        passwordVariable: 'NEXUS_PASS')]) {
        sh '''
          # write settings file (unquoted heredoc so shell expands $NEXUS_USER/$NEXUS_PASS)
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

          # debug: show file exists (do not print contents with credentials)
          echo "ci-settings.xml created: $(ls -la ci-settings.xml || true)"

          # run deploy with debug so upload lines are visible; tee to capture output
          mvn -B -DskipTests -X deploy --settings ci-settings.xml | tee mvn-deploy.log
          tail -n 200 mvn-deploy.log || true
        '''
      }

      // cleanup
      sh 'rm -f ci-settings.xml mvn-deploy.log || true'
    }
  }
}
