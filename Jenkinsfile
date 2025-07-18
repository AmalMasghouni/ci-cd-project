pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://localhost:9000'
        SONAR_TOKEN = credentials('sonar-token')
        NEXUS_URL = 'http://localhost:8081'
        NEXUS_CREDENTIALS = 'nexus-admin-credentials'
        GIT_CREDENTIALS = 'AmalMasghouni'
        VERSION = "${env.BUILD_NUMBER}"
        JAR_FILE = "target/Foyer-${VERSION}.jar"
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
                checkout scm: [
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/AmalMasghouni/ci-cd-project.git',
                        credentialsId: env.GIT_CREDENTIALS
                    ]]
                ]
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
                withSonarQubeEnv('SonarQube') {
                    sh "mvn sonar:sonar -Dsonar.projectKey=alinfo5-groupe4-2 -Dsonar.host.url=${env.SONAR_HOST_URL} -Dsonar.login=${env.SONAR_TOKEN}"
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
                    }
                }
            }
        }

        stage('Package') {
            when {
                expression { env.QUALITY_GATE_STATUS == 'OK' }
            }
            steps {
                echo "📦 Packaging the application..."
                sh 'mvn package -DskipTests'
                sh "cp target/Foyer-0.0.1.jar ${JAR_FILE}"
            }
        }

        stage('Upload to Nexus') {
            when {
                expression { env.QUALITY_GATE_STATUS == 'OK' }
            }
            steps {
                echo "⬆️ Uploading to Nexus..."
                nexusArtifactUploader(
                    nexusUrl: env.NEXUS_URL,
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    repository: 'Foyer',
                    credentialsId: env.NEXUS_CREDENTIALS,
                    groupId: 'com.example',
                    version: env.VERSION,
                    artifacts: [[
                        artifactId: 'Foyer',
                        classifier: '',
                        file: env.JAR_FILE,
                        type: 'jar'
                    ]]
                )
            }
        }
    }

    post {
        success {
            echo '🎉 Build completed successfully!'
            script {
                if (env.QUALITY_GATE_STATUS == 'OK') {
                    emailext (
                        subject: "SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                        body: """<p>🎉 Build Successfully Deployed!</p>
                                 <p>Project: ${env.JOB_NAME}</p>
                                 <p>Build: #${env.BUILD_NUMBER}</p>
                                 <p>Quality Gate: PASSED ✅</p>
                                 <p>Artifact: Foyer-${env.BUILD_NUMBER}.jar</p>
                                 <p><a href="${env.BUILD_URL}">View Build</a></p>""",
                        to: 'masghouniamal84@gmail.com',
                        mimeType: 'text/html'
                    )
                } else {
                    emailext (
                        subject: "WARNING: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                        body: """<p>⚠️ Quality Gate Failed</p>
                                 <p>Build succeeded but artifacts NOT deployed</p>
                                 <p>Status: ${env.QUALITY_GATE_STATUS}</p>
                                 <p><a href="${env.BUILD_URL}">Investigate Build</a></p>""",
                        to: 'masghouniamal84@gmail.com',
                        mimeType: 'text/html'
                    )
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
                    to: 'masghouniamal84@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
        always {
            echo '🧹 Cleaning up...'
        }
    }
}