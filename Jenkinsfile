pipeline {
    agent any

    parameters {
        choice(
            name: 'ENV',
            choices: ['dev', 'qa', 'prod'],
            description: 'Select target environment'
        )
    }

    environment {
        REPO_URL = 'https://github.com/ThanujaRatakonda/kp_9'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: "${REPO_URL}"
            }
        }

        stage('Prepare Manifests') {
            steps {
                sh '''
                echo "Using ENV=${ENV}"

                mkdir -p rendered/argocd
                mkdir -p rendered/k8s

                # Replace ENV in ArgoCD apps
                for f in argocd/*.yaml; do
                  envsubst < $f > rendered/argocd/$(basename $f)
                done

                # Replace ENV in infra files
                for f in k8s/*.yaml; do
                  envsubst < $f > rendered/k8s/$(basename $f)
                done
                '''
            }
        }

        stage('Deploy Infra (Namespace + PV + PVC)') {
            steps {
                sh '''
                kubectl apply -f rendered/k8s/
                '''
            }
        }

        stage('Deploy Applications via ArgoCD') {
            steps {
                sh '''
                kubectl apply -f rendered/argocd/
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment to ${ENV} completed successfully "
        }
        failure {
            echo "Deployment to ${ENV} failed "
        }
    }
}
