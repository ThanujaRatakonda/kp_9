pipeline {
    agent any

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_9"
        TRIVY_OUTPUT_JSON = "trivy-output.json"
        K8S_NAMESPACE = "dev"
    }

    parameters {
        choice(
            name: 'ACTION',
            choices: ['FULL_PIPELINE', 'FRONTEND_ONLY', 'BACKEND_ONLY'],
            description: 'Choose FULL_PIPELINE, FRONTEND_ONLY, or BACKEND_ONLY'
        )
    }

    stages {
        stage('Checkout') {
            when { expression { params.ACTION != 'SCALE_ONLY' } }
            steps {
                git 'https://github.com/ThanujaRatakonda/kp_9.git'
            }
        }

        // ---------------- FRONTEND ----------------
        stage('Build Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                sh "docker build -t frontend:${IMAGE_TAG} ./frontend"
            }
        }

        stage('Scan Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                sh """
                    trivy image frontend:${IMAGE_TAG} \
                    --severity CRITICAL,HIGH \
                    --format json -o ${TRIVY_OUTPUT_JSON}
                """
                archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true
                script {
                    def vulnerabilities = sh(script: """
                        jq '[.Results[] |
                             (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                             (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                            ] | length' ${TRIVY_OUTPUT_JSON}
                    """, returnStdout: true).trim()
                    if (vulnerabilities.toInteger() > 0) {
                        error "CRITICAL/HIGH vulnerabilities found in frontend!"
                    }
                }
            }
        }

        stage('Push Frontend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'FRONTEND_ONLY'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"
                        sh "docker tag frontend:${IMAGE_TAG} ${fullImage}"
                        sh "docker push ${fullImage}"
                        sh "docker rmi frontend:${IMAGE_TAG} || true"
                    }
                }
            }
        }

        // ---------------- BACKEND ----------------
        stage('Build Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                sh "docker build -t backend:${IMAGE_TAG} ./backend"
            }
        }

        stage('Scan Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                sh """
                    trivy image backend:${IMAGE_TAG} \
                    --severity CRITICAL,HIGH \
                    --format json -o ${TRIVY_OUTPUT_JSON}
                """
                archiveArtifacts artifacts: "${TRIVY_OUTPUT_JSON}", fingerprint: true
                script {
                    def vulnerabilities = sh(script: """
                        jq '[.Results[] |
                             (.Packages // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH")) +
                             (.Vulnerabilities // [] | .[]? | select(.Severity=="CRITICAL" or .Severity=="HIGH"))
                            ] | length' ${TRIVY_OUTPUT_JSON}
                    """, returnStdout: true).trim()
                    if (vulnerabilities.toInteger() > 0) {
                        error "CRITICAL/HIGH vulnerabilities found in backend!"
                    }
                }
            }
        }

        stage('Push Backend') {
            when { expression { params.ACTION in ['FULL_PIPELINE', 'BACKEND_ONLY'] } }
            steps {
                script {
                    def fullImage = "${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
                        sh "echo \$HARBOR_PASS | docker login ${HARBOR_URL} -u \$HARBOR_USER --password-stdin"
                        sh "docker tag backend:${IMAGE_TAG} ${fullImage}"
                        sh "docker push ${fullImage}"
                        sh "docker rmi backend:${IMAGE_TAG} || true"
                    }
                }
            }
        }

        // ---------------- Deploy to Kubernetes ----------------
        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f k8s/ -n ${K8S_NAMESPACE}"
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                // Trigger ArgoCD sync using CLI
                sh """
                argocd app sync frontend --server https://argocd.example.com --auth-token \$ARGOCD_TOKEN || true
                argocd app sync backend --server https://argocd.example.com --auth-token \$ARGOCD_TOKEN || true
                """
            }
        }
    }
}
