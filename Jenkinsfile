/* groovylint-disable CompileStatic, GStringExpressionWithinString */
pipeline {
    agent any

    tools {
        maven 'MAVEN3.9'
        jdk 'JDK17'
    }

    environment {
        SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin123'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUSIP = '172.31.22.72'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN = 'nexuslogin'
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo 'Now Archiving.'
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -s settings.xml test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // parameter indicates whether to set pipeline to UNSTABLE
                    // true = set pipeline to UNSTABLE false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('UploadArtifacts') {
            steps {
                    nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: "${RELEASE_REPO}",
                    credentialsId: "${NEXUS_LOGIN}",
                    artifacts: [
                        [artifactId: 'vproapp',
                        classifier: '',
                        file: 'target/vprofile-v2.war',
                        type: 'war']
                    ]
                )
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution completed'
        }
        success {
            slackSend(
                channel: '#jenkinscicd',
                color: 'good',
                message: '*BUILD SUCCESS*\n' +
                        "*Job:* ${env.JOB_NAME}\n" +
                        "*Build:* #${env.BUILD_NUMBER}\n" +
                        "*Branch:* ${env.GIT_BRANCH}\n" +
                        "*Commit:* ${env.GIT_COMMIT.take(8)}\n" +
                        "*Duration:* ${currentBuild.durationString}\n" +
                        '*Artifacts:* Uploaded to Nexus\n' +
                        '*SonarQube:* Quality Gate PASSED'
            )
        }
        failure {
            slackSend(
                channel: '#jenkinscicd',
                color: 'danger',
                message: '*BUILD FAILED*\n' +
                        "*Job:* ${env.JOB_NAME}\n" +
                        "*Build:* #${env.BUILD_NUMBER}\n" +
                        "*Branch:* ${env.GIT_BRANCH}\n" +
                        "*Commit:* ${env.GIT_COMMIT.take(8)}\n" +
                        "*Duration:* ${currentBuild.durationString}\n" +
                        "*Check:* ${env.BUILD_URL}console"
            )
        }
        unstable {
            slackSend(
                channel: '#jenkinscicd',
                color: 'warning',
                message: '*BUILD UNSTABLE*\n' +
                        "*Job:* ${env.JOB_NAME}\n" +
                        "*Build:* #${env.BUILD_NUMBER}\n" +
                        '*Quality Gate may have issues*'
            )
        }
    }
}
