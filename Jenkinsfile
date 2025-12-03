pipeline {
    agent any

    environment {
        // ==== GCP CONFIG ====
        PROJECT_ID   = 'medical-rag-480120'
        REGION       = 'us-central1'                // change if you used another region
        REPOSITORY   = 'medical-assistant-repo'     // Artifact Registry repo you created
        SERVICE_NAME = 'medical-assistant'          // Cloud Run service name
        GCP_CRED_ID  = 'gcp-jenkins-sa'             // Jenkins credentials ID for your JSON key

        // ==== IMAGE TAG ====
        IMAGE_TAG    = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Clone GitHub Repo') {
            steps {
                script {
                    echo 'Cloning GitHub repo to Jenkins...'
                    checkout scmGit(
                        branches: [[name: '*/main']],
                        userRemoteConfigs: [[
                            url: 'https://github.com/fatirmalik1/agentic-medical-assistant.git'
                        ]]
                    )
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    def imageName = "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}/${SERVICE_NAME}:${IMAGE_TAG}"
                    sh """
                        docker build -t ${imageName} .
                    """
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    echo 'Running Trivy scan on Docker image...'
                    def imageName = "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}/${SERVICE_NAME}:${IMAGE_TAG}"
                    sh """
                        trivy image --exit-code 0 --severity HIGH,CRITICAL ${imageName}
                        # change --exit-code 0 to --exit-code 1 if you want to fail on vulnerabilities
                    """
                }
            }
        }

        stage('Authenticate to GCP & Push Image') {
            steps {
                withCredentials([
                    file(credentialsId: "${GCP_CRED_ID}", variable: 'GOOGLE_APPLICATION_CREDENTIALS_FILE')
                ]) {
                    script {
                        echo 'Authenticating to GCP and pushing image to Artifact Registry...'
                        def imageName = "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}/${SERVICE_NAME}:${IMAGE_TAG}"

                        sh """
                            # Activate service account
                            gcloud auth activate-service-account \
                              --key-file=${GOOGLE_APPLICATION_CREDENTIALS_FILE}

                            # Set project and region
                            gcloud config set project ${PROJECT_ID}
                            gcloud config set run/region ${REGION}

                            # Configure Docker to use Artifact Registry
                            gcloud auth configure-docker ${REGION}-docker.pkg.dev -q

                            # Push the image
                            docker push ${imageName}
                        """
                    }
                }
            }
        }

        stage('Deploy to Cloud Run') {
            steps {
                withCredentials([
                    file(credentialsId: "${GCP_CRED_ID}", variable: 'GOOGLE_APPLICATION_CREDENTIALS_FILE')
                ]) {
                    script {
                        echo 'Deploying to Cloud Run...'
                        def imageName = "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}/${SERVICE_NAME}:${IMAGE_TAG}"

                        sh """
                            # Authenticate again in case this runs on a different agent
                            gcloud auth activate-service-account \
                              --key-file=${GOOGLE_APPLICATION_CREDENTIALS_FILE}

                            gcloud config set project ${PROJECT_ID}
                            gcloud config set run/region ${REGION}

                            # Deploy to Cloud Run
                            gcloud run deploy ${SERVICE_NAME} \
                              --image=${imageName} \
                              --platform=managed \
                              --region=${REGION} \
                              --allow-unauthenticated
                        """
                    }
                }
            }
        }
    }
}