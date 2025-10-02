pipeline {
    agent any
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        PROJECT_ID = 'cicd-2024'
        REGION = 'asia-south2'
        REPO_NAME = 'bankapp'
        IMAGE_NAME = 'bankapp'
        IMAGE_TAG = 'latest'
        ARTIFACT_REGISTRY = "${REGION}-docker.pkg.dev"
        FULL_IMAGE = "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        stage('git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/devopsgcp/bankingapp.git'
            }
        }

        stage('trivy fs scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL --format json -o trivy-scan-report.json .'
            }
        }

        stage('Sonarqube-scanner') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=bankapp \
                        -Dsonar.projectName=bankapp -Dsonar.sources=. \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('Quality gate check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('mvn package') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Push_to_nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'bankapp', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('gcp_authentication') {
            steps {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        echo "Activating service account"
                        gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                        gcloud config set project $PROJECT_ID
                        gcloud auth configure-docker $ARTIFACT_REGISTRY --quiet
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $ARTIFACT_REGISTRY/$PROJECT_ID/$REPO_NAME/$IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Trivy Scan (Docker Image)') {
            steps {
                sh '''
                    echo "Scanning Docker image with Trivy..."
                    trivy image --severity HIGH,CRITICAL --format json -o trivy-image-report.json $FULL_IMAGE
                '''
            }
        }

        stage('Push to Artifact Registry') {
            steps {
                sh '''
                    docker push $ARTIFACT_REGISTRY/$PROJECT_ID/$REPO_NAME/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

    } // end of stages

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'krsindia69@gmail.com',
                    from: 'kumarindia528@gmail.com',
                    replyTo: 'kumarindia528@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
