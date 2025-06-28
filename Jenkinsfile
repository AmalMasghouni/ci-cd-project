pipeline {
    agent any
    triggers { githubPush() }

    environment {
        SONAR_HOST_URL = 'http://192.168.33.10:9000'
        SONAR_TOKEN = credentials('SONAR_TOKEN2')
        VERSION = "${env.BUILD_NUMBER}"
        JAR_FILE = "target/Foyer-${VERSION}.jar"
    }

    options {
        skipStagesAfterUnstable()
        timestamps()
    }

    stages {
        stage('Preparation') {
            steps {
                echo "🔄 Cleaning workspace..."
                deleteDir()
            }
        }

        stage('Checkout') {
            steps {
                echo "📥 Cloning repository..."
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                echo "🔧 Compiling and testing..."
                sh 'mvn compile test -Dspring.profiles.active=test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('MySonarQubeServer') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=alinfo5-groupe4-2'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        echo "Quality Gate status: ${qg.status}"
                        env.QUALITY_GATE_STATUS = qg.status
                        echo "Quality Gate status: ${env.QUALITY_GATE_STATUS}"
                    }
                }
            }
        }

        stage('Package') {
            when { expression { env.QUALITY_GATE_STATUS == 'OK' } }
            steps {
                echo "📦 Packaging the application..."
                sh 'mvn package -DskipTests'
                sh "cp target/Foyer-0.0.1.jar ${JAR_FILE}"
            }
        }

        stage('Upload to Nexus') {
            when { expression { env.QUALITY_GATE_STATUS == 'OK' } }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    echo "⬆️ Uploading to Nexus..."
                    nexusArtifactUploader(
                        artifacts: [[
                            artifactId: 'Foyer',
                            classifier: '',
                            file: env.JAR_FILE,
                            type: 'jar'
                        ]],
                        credentialsId: 'nexus-creds',
                        groupId: 'com.example',
                        nexusUrl: '192.168.33.10:8081',
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        repository: 'Foyer',
                        version: env.VERSION
                    )
                }
            }
        }
    }

    post {
        success {
            echo '🎉 Build completed successfully!'
            script {
                script {
                    if (env.QUALITY_GATE_STATUS == 'OK') {
                        emailext (
                            subject: "SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                            body: """<p>🎉 <b>Build Successfully Deployed!</b></p>
                                     <p>Project: ${env.JOB_NAME}</p>
                                     <p>Build: #${env.BUILD_NUMBER}</p>
                                     <p>Quality Gate: PASSED ✅</p>
                                     <p>Artifact: Foyer-${env.BUILD_NUMBER}.jar</p>
                                     <p><a href="${env.BUILD_URL}">View Build</a></p>""",
                            to: 'devops@example.com',
                            mimeType: 'text/html'
                        )
                    } else {
                        emailext (
                            subject: "WARNING: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                            body: """<p>⚠️ <b>Quality Gate Failed</b></p>
                                     <p>Build succeeded but artifacts NOT deployed</p>
                                     <p>Status: ${env.QUALITY_GATE_STATUS}</p>
                                     <p><a href="${env.BUILD_URL}">Investigate Build</a></p>""",
                            to: 'devops@example.com',
                            mimeType: 'text/html'
                        )
                    }
                }
            }
        }
        failure {
            echo '💥 Build failed.'
            script {
                emailext (
                    subject: "FAILED: Job ${env.JOB_NAME} - Build ${env.BUILD_NUMBER}",
                    body: """<p>❌ Build failed during stage: ${currentBuild.currentResult}</p>
                             <p>Quality Gate Status: ${env.QUALITY_GATE_STATUS ?: 'N/A'}</p>
                             <p>Check console output: <a href="${env.BUILD_URL}">${env.JOB_NAME} #${env.BUILD_NUMBER}</a></p>""",
                    to: 'dev-team@example.com',
                    mimeType: 'text/html'
                )
            }
        }
        always {
            echo '🧹 Cleaning up...'
        }
    }
}