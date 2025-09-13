pipeline {
    agent any

    tools {
        jdk "JDK17"
        maven "MAVEN3.9"
    }

    environment {
        MAVEN_SETTINGS = 'settings.xml'
        SONAR_SCANNER = tool 'sonarscanner4'

        NEXUS_URL = 'http://172.31.26.78'
        NEXUS_CREDENTIALS = 'Nexus-Credentials'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin123'

        NEXUS_REPO = 'nexus-hosted-artifact'           // Change to 'maven-snapshots' if needed
        NEXUSPORT = '8081'
        NEXUS_PROXY = 'nexus-proxy-repo'


        GROUP_ID = 'QA'
        ARTIFACT_ID = 'vprofileMahmmod'
        

    }

    options {
        timestamps()
        ansiColor('xterm')
        timeout(time: 60, unit: 'MINUTES')
    }

    stages {

        stage('BUILD') {
            steps {
                sh "mvn -s ${MAVEN_SETTINGS} clean install -DskipTests"
            }
            post {
                success {
                    echo 'Build successful, now archiving artifacts...'
                    archiveArtifacts artifacts: 'target/*.war', fingerprint: true
                }
                failure {
                    echo 'Build failed!'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh "mvn -s ${MAVEN_SETTINGS} test"
            }
            post {
                success {
                    junit 'target/surefire-reports/*.xml'
                }
                failure {
                    echo 'Unit tests failed!'
                }
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh "mvn -s ${MAVEN_SETTINGS} verify -DskipUnitTests"
            }
            post {
                success {
                    echo 'Integration tests passed!'
                }
                failure {
                    echo 'Integration tests failed!'
                }
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh "mvn -s ${MAVEN_SETTINGS} checkstyle:checkstyle"
            }
            post {
                success {
                    echo 'Checkstyle report generated.'
                    archiveArtifacts artifacts: 'target/checkstyle-result.xml', allowEmptyArchive: true
                }
                failure {
                    echo 'Checkstyle failed!'
                }
            }
        }

        stage('CODE ANALYSIS WITH SONARQUBE') {
            environment {
                SCANNER_HOME = tool 'sonarscanner4'
                SONAR_PROJECT_KEY = 'ci-project'
                SONAR_PROJECT_NAME = 'ci-project'
                SONAR_PROJECT_VERSION = '1.0'
                SONAR_SOURCES = 'src/'
                SONAR_BINARIES = 'target/classes/'
                SONAR_JUNIT_REPORTS = 'target/surefire-reports/'
                SONAR_JACOCO_REPORTS = 'target/jacoco.exec'
                SONAR_CHECKSTYLE_REPORT = 'target/checkstyle-result.xml'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                        -Dsonar.projectVersion=${SONAR_PROJECT_VERSION} \
                        -Dsonar.sources=${SONAR_SOURCES} \
                        -Dsonar.java.binaries=${SONAR_BINARIES} \
                        -Dsonar.junit.reportsPath=${SONAR_JUNIT_REPORTS} \
                        -Dsonar.jacoco.reportPaths=${SONAR_JACOCO_REPORTS} \
                        -Dsonar.java.checkstyle.reportPaths=${SONAR_CHECKSTYLE_REPORT}
                    """
                }
                
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
            post {
                success {
                    echo 'SonarQube analysis completed.'
                }
                failure {
                    echo 'SonarQube analysis failed or Quality Gate not passed.'
                }
            }
        }



        stage('Upload Artifacts to Nexus') {

            steps {
                script {
                    // Generate version dynamically using BUILD_NUMBER
                    def artifactVersion = "1.0.${env.BUILD_NUMBER}"

                    // Find the WAR file in target dynamically
                    def warFile = sh(script: "ls target/*.war | head -n 1", returnStdout: true).trim()

                    // Rename it with BUILD_NUMBER version
                    sh "mv ${warFile} target/${ARTIFACT_ID}-${artifactVersion}.war"

                    // Upload artifact to Nexus
                    nexusArtifactUploader artifacts: [
                        [
                            artifactId: "${ARTIFACT_ID}",
                            classifier: '',
                            file: "target/${ARTIFACT_ID}-${artifactVersion}.war",
                            type: 'war'
                        ]
                    ],
                    credentialsId: "${NEXUS_CREDENTIALS}",
                    groupId: "${GROUP_ID}",
                    nexusUrl: "${NEXUS_URL}",
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    repository: "${NEXUS_REPO}",
                    version: artifactVersion
                }
            }
            post {
                success {
                    echo "Artifact ${ARTIFACT_ID}-${artifactVersion}.war uploaded to Nexus successfully."
                }
                failure {
                    echo "Failed to upload artifact to Nexus."
                }
            }
        }

    }

    post {
        always {
            slackSend (
                channel: '#jenkins-channel', // Specify your Slack channel
                message: "Pipeline ${currentBuild.fullDisplayName} finished with status: ${currentBuild.currentResult}",
                color: "${currentBuild.currentResult == 'SUCCESS' ? 'good' : 'danger'}"
            )
        }
    }
}
