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
        IMAGE_TAG = "${BUILD_NUMBER}"  // Unique for each build
        ARTIFACT_REGISTRY = "${REGION}-docker.pkg.dev"
        FULL_IMAGE = "${ARTIFACT_REGISTRY}/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/devopsgcp/bankingapp.git'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL --format json -o trivy-fs-report.json . || exit 1'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=bankapp \
                        -Dsonar.projectName=bankapp \
                        -Dsonar.sources=. \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Push to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'bankapp', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('GCP Authentication') {
            steps {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
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
                    docker build -t $FULL_IMAGE .
                '''
            }
        }

        stage('Trivy Docker Scan') {
            steps {
                sh '''
                    trivy image --severity HIGH,CRITICAL --format json -o trivy-image-report.json $FULL_IMAGE || exit 1
                '''
            }
        }

        stage('Push Docker Image to Artifact Registry') {
            steps {
                sh '''
                    docker push $FULL_IMAGE
                '''
            }
        }

        stage('Deploy to GKE') {
            steps {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                        gcloud config set project $PROJECT_ID
                        export USE_GKE_GCLOUD_AUTH_PLUGIN=True
                        gcloud container clusters get-credentials bankapp --region=$REGION --project=$PROJECT_ID
                        
                        # Apply Kubernetes manifests
                       # kubectl get namespace webapps || kubectl create namespace webapps
                       # kubectl apply  -f k8s/RBAC/
                        kubectl delete -n webapps -f k8s/
                        kubectl apply -f k8s/

                    
                    '''
                }
            }
        }
    }

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
