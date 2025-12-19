pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_7"
    }

    parameters {
        choice(
            name: 'SERVICE',
            choices: ['all', 'frontend', 'backend'],
            description: 'Which service to build'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ThanujaRatakonda/kp_7.git'
            }
        }

        stage('Build & Push Backend') {
            when { expression { params.SERVICE in ['all', 'backend'] } }
            steps {
                sh """
                docker build -t backend:${IMAGE_TAG} backend
                docker tag backend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                docker push ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                """
            }
        }

        stage('Build & Push Frontend') {
            when { expression { params.SERVICE in ['all', 'frontend'] } }
            steps {
                sh """
                docker build -t frontend:${IMAGE_TAG} frontend
                docker tag frontend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                docker push ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                """
            }
        }

        stage('Update Helm values') {
            steps {
                script {
                    if (params.SERVICE in ['all', 'backend']) {
                        sh """
                        sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' backend-hc/values.yaml
                        """
                    }

                    if (params.SERVICE in ['all', 'frontend']) {
                        sh """
                        sed -i 's/tag:.*/tag: "${IMAGE_TAG}"/' frontend-hc/values.yaml
                        """
                    }
                }
            }
        }

        stage('Commit & Push to Git') {
            steps {
                sh """
               // git config user.name "jenkins"
                git config user.email "jenkins@local"
                git add .
                git commit -m "Update image tag to ${IMAGE_TAG}"
                git push origin master
                """
            }
        }
    }
}
