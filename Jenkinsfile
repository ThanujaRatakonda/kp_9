pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        HARBOR_URL = "10.131.103.92:8090"
        HARBOR_PROJECT = "kp_9"
    }

    parameters {
        choice(
            name: 'SERVICE',
            choices: ['all', 'frontend', 'backend'],
            description: 'Which service to build and push'
        )
        string(name: 'ENV', defaultValue: 'dev', description: 'Namespace/Environment to deploy')
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ThanujaRatakonda/kp_9.git'
            }
        }

        stage('Build & Push Backend') {
            when { expression { params.SERVICE in ['all', 'backend'] } }
            steps {
                sh """
                docker build -t backend:${IMAGE_TAG} backend
                docker tag backend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'H_USER', passwordVariable: 'H_PASS')]) {
                    echo \$H_PASS | docker login ${HARBOR_URL} -u \$H_USER --password-stdin
                    docker push ${HARBOR_URL}/${HARBOR_PROJECT}/backend:${IMAGE_TAG}
                }
                """
            }
        }

        stage('Build & Push Frontend') {
            when { expression { params.SERVICE in ['all', 'frontend'] } }
            steps {
                sh """
                docker build -t frontend:${IMAGE_TAG} frontend
                docker tag frontend:${IMAGE_TAG} ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'H_USER', passwordVariable: 'H_PASS')]) {
                    echo \$H_PASS | docker login ${HARBOR_URL} -u \$H_USER --password-stdin
                    docker push ${HARBOR_URL}/${HARBOR_PROJECT}/frontend:${IMAGE_TAG}
                }
                """
            }
        }

        stage('Update Helm values') {
            steps {
                script {
                    if (params.SERVICE in ['all', 'backend']) {
                        sh "sed -i 's/tag:.*/tag: \"${IMAGE_TAG}\"/' backend-hc/backendvalues.yaml"
                    }
                    if (params.SERVICE in ['all', 'frontend']) {
                        sh "sed -i 's/tag:.*/tag: \"${IMAGE_TAG}\"/' frontend-hc/frontendvalues.yaml"
                    }
                }
            }
        }

        stage('Deploy K8s & ArgoCD') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                    export ENV=${params.ENV}

                    # Apply k8s manifests
                    for file in k8s/*; do
                        envsubst < \$file | kubectl apply -n \$ENV -f -
                    done

                    # Apply argocd manifests (keep argocd namespace as is)
                    for file in argocd/*; do
                        ns_argocd=$(grep -E 'namespace:' \$file | head -1 | awk '{print \$2}')
                        if [ "\$ns_argocd" = "argocd" ]; then
                            kubectl apply -f \$file
                        else
                            envsubst < \$file | kubectl apply -n \$ENV -f -
                        fi
                    done
                    """
                }
            }
        }

        stage('Commit & Push to Git') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh """
                    git config user.name "thanuja"
                    git config user.email "ratakondathanuja@gmail.com"
                    git add .
                    git commit -m "Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"
                    git push https://${GIT_USER}:${GIT_TOKEN}@github.com/ThanujaRatakonda/kp_9.git master
                    """
                }
            }
        }
    }
}
