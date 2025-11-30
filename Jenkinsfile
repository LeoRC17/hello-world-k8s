pipeline {
    agent any
    
    tools {
        jdk 'jdk'
        maven 'maven'
    }
    
    environment {
        NEXUS_URL = 'http://192.168.49.2:30001/repository/maven-releases/'  // Update with your Nexus URL
        NEXUS_REPOSITORY = 'maven-releases'          // or 'maven-snapshots'
        NEXUS_CREDENTIALS_ID = 'nexus-creds'   // Jenkins credentials ID for Nexus
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
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
                script {
                    sh 'mvn -B clean package -DskipTests'
                }
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
                    // Parse pom.xml directly (safe approach without script approvals)
                    sh '''
                        ARTIFACT_VERSION=$(grep -oPm1 "(?<=<version>)[^<]+" pom.xml | head -1)
                        ARTIFACT_ID=$(grep -oPm1 "(?<=<artifactId>)[^<]+" pom.xml | head -1)
                        GROUP_ID=$(grep -oPm1 "(?<=<groupId>)[^<]+" pom.xml | head -1)
                        
                        echo "Resolved: version=${ARTIFACT_VERSION}, artifactId=${ARTIFACT_ID}, groupId=${GROUP_ID}"
                        
                        # Write to files for later use
                        echo "${ARTIFACT_VERSION}" > version.txt
                        echo "${ARTIFACT_ID}" > artifactId.txt
                        echo "${GROUP_ID}" > groupId.txt
                    '''
                    
                    // Read from files and set environment variables
                    env.ARTIFACT_VERSION = readFile('version.txt').trim()
                    env.ARTIFACT_ID = readFile('artifactId.txt').trim()
                    env.GROUP_ID = readFile('groupId.txt').trim()
                    
                    echo "Resolved artifact metadata:"
                    echo "Version: ${ARTIFACT_VERSION}"
                    echo "ArtifactId: ${ARTIFACT_ID}"
                    echo "GroupId: ${GROUP_ID}"
                }
            }
        }
        
        stage('Push to Nexus') {
            when {
                expression { env.ARTIFACT_VERSION && env.ARTIFACT_ID && env.GROUP_ID }
            }
            steps {
                script {
                    echo "Pushing ${GROUP_ID}:${ARTIFACT_ID}:${ARTIFACT_VERSION} to Nexus"
                    
                    withCredentials([usernamePassword(
                        credentialsId: env.NEXUS_CREDENTIALS_ID,
                        passwordVariable: 'NEXUS_PASSWORD',
                        usernameVariable: 'NEXUS_USERNAME'
                    )]) {
                        // Deploy to Nexus using Maven
                        sh """
                            mvn -B deploy:deploy-file \
                                -DgroupId=${GROUP_ID} \
                                -DartifactId=${ARTIFACT_ID} \
                                -Dversion=${ARTIFACT_VERSION} \
                                -Dpackaging=jar \
                                -Dfile=target/${ARTIFACT_ID}-${ARTIFACT_VERSION}.jar \
                                -DrepositoryId=nexus \
                                -Durl=${NEXUS_URL}/repository/${NEXUS_REPOSITORY} \
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
            // Cleanup
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
