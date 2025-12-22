pipeline {
    agent any

    environment {
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_9"
        IMAGE_TAG = "${BUILD_NUMBER}"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
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
                git url: 'https://github.com/ThanujaRatakonda/kp_9.git', branch: 'master'
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    if (params.SERVICE == 'all' || params.SERVICE == 'backend') {
                        sh "docker build -t ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG} ./backend"
                    }
                    if (params.SERVICE == 'all' || params.SERVICE == 'frontend') {
                        sh "docker build -t ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG} ./frontend"
                    }
                }
            }
        }

        stage('Scan Images with Trivy') {
            steps {
                script {
                    if (params.SERVICE == 'all' || params.SERVICE == 'backend') {
                        sh "trivy image --format json -o ${TRIVY_OUTPUT_JSON} ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
                    }
                    if (params.SERVICE == 'all' || params.SERVICE == 'frontend') {
                        sh "trivy image --format json -o ${TRIVY_OUTPUT_JSON} ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    if (params.SERVICE == 'all' || params.SERVICE == 'backend') {
                        sh "docker push ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
                    }
                    if (params.SERVICE == 'all' || params.SERVICE == 'frontend') {
                        sh "docker push ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Deploy via ArgoCD') {
            steps {
                script {
                    if (params.SERVICE == 'all' || params.SERVICE == 'backend') {
                        sh "argocd app sync backend --refresh"
                    }
                    if (params.SERVICE == 'all' || params.SERVICE == 'frontend') {
                        sh "argocd app sync frontend --refresh"
                    }
                    // Database sync is always done because it's a PostgreSQL chart
                    if (params.SERVICE == 'all') {
                        sh "argocd app sync database --refresh"
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh "kubectl get pods -n \${ENV}"
                sh "kubectl get svc -n \${ENV}"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
