pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "blurryface027/prime-video-devsecops"
        IMAGE_TAG = "${BUILD_NUMBER}"
        GITOPS_REPO = "https://github.com/blurryface027/prime-video-gitops.git"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/blurryface027/amazon-prime-devsecops.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=Prime-Video-Clone \
                    -Dsonar.projectName=Prime-Video-Clone \
                    -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs .'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Docker Scout Scan') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                    echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin

                    docker scout quickview ${IMAGE_NAME}:${IMAGE_TAG}
                    docker scout cves ${IMAGE_NAME}:${IMAGE_TAG} || true

                    docker logout
                    """
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                    echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest

                    docker logout
                    """
                }
            }
        }

        stage('Update GitOps Repository') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-cred',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh """
                    rm -rf prime-video-gitops

                    git clone https://\$GIT_USER:\$GIT_TOKEN@github.com/blurryface027/prime-video-gitops.git

                    cd prime-video-gitops

                    sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' prime-video-chart/values.yaml

                    git config user.name "Jenkins"
                    git config user.email "jenkins@local"

                    git add prime-video-chart/values.yaml

                    git diff --cached --quiet || git commit -m "Update image tag to ${IMAGE_TAG}"

                    git push origin main
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed."
        }

        always {
            cleanWs()
        }
    }
}