
pipeline {
  agent any

  parameters {
    string(name: 'NAMESPACE', defaultValue: 'dev', description: 'Target Kubernetes namespace')
  }

  environment {
    REGISTRY = '10.131.103.92:8090'
    PROJECT  = 'kp_9'
    IMAGE_TAG = "${env.BUILD_NUMBER}"

    // Trivy
    TRIVY_SEVERITY = "CRITICAL,HIGH"
    TRIVY_IGNORE_UNFIXED = "true"  // set to "false" if you want to count unfixed vulns too
  }

  stages {

    stage('Checkout') {
      steps {
        // ✅ Only checkout your current repo (kp_9). Remove extra git checkout to kp_6.
        checkout scm
      }
    }

    stage('Docker Login (Harbor)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')]) {
          sh '''
            set -e
            echo "$HARBOR_PASS" | docker login ${REGISTRY} -u "$HARBOR_USER" --password-stdin
          '''
        }
      }
    }

    /* ===========================
       Backend: Build → Scan → Push
       =========================== */
    stage('Build Backend Image') {
      steps {
        sh '''
          set -e
          docker build -t ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} backend/
        '''
      }
    }

    stage('Trivy Scan Backend') {
      steps {
        sh '''
          set -e
          mkdir -p reports

          IGNORE_FLAG=""
          if [ "${TRIVY_IGNORE_UNFIXED}" = "true" ]; then
            IGNORE_FLAG="--ignore-unfixed"
          fi

          # Make reports
          trivy image ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format table -o reports/backend_${IMAGE_TAG}_trivy.txt || true

          trivy image ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format json -o reports/backend_${IMAGE_TAG}_trivy.json || true

          trivy image ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format sarif -o reports/backend_${IMAGE_TAG}_trivy.sarif || true

          # Gate (fail if HIGH/CRITICAL) — use Trivy exit code for simplicity
          trivy image ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --exit-code 1 --format table
        '''
        archiveArtifacts artifacts: "reports/backend_${IMAGE_TAG}_trivy.*", fingerprint: true, allowEmptyArchive: false
      }
    }

    stage('Push Backend Image') {
      steps {
        sh '''
          set -e
          docker push ${REGISTRY}/${PROJECT}/backend:${IMAGE_TAG}
        '''
      }
    }

    /* ===========================
       Frontend: Build → Scan → Push
       =========================== */
    stage('Build Frontend Image') {
      steps {
        sh '''
          set -e
          docker build -t ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} frontend/
        '''
      }
    }

    stage('Trivy Scan Frontend') {
      steps {
        sh '''
          set -e
          mkdir -p reports

          IGNORE_FLAG=""
          if [ "${TRIVY_IGNORE_UNFIXED}" = "true" ]; then
            IGNORE_FLAG="--ignore-unfixed"
          fi

          # Make reports
          trivy image ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format table -o reports/frontend_${IMAGE_TAG}_trivy.txt || true

          trivy image ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format json -o reports/frontend_${IMAGE_TAG}_trivy.json || true

          trivy image ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --format sarif -o reports/frontend_${IMAGE_TAG}_trivy.sarif || true

          # Gate (fail if HIGH/CRITICAL)
          trivy image ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG} \
            --severity ${TRIVY_SEVERITY} ${IGNORE_FLAG} \
            --exit-code 1 --format table
        '''
        archiveArtifacts artifacts: "reports/frontend_${IMAGE_TAG}_trivy.*", fingerprint: true, allowEmptyArchive: false
      }
    }

    stage('Push Frontend Image') {
      steps {
        sh '''
          set -e
          docker push ${REGISTRY}/${PROJECT}/frontend:${IMAGE_TAG}
        '''
      }
    }

    /* ===========================
       K8s: Namespace, regcred, base SC/PV/PVC
       =========================== */
    stage('Prepare Kube Access + Namespace + regcred') {
      steps {
        withCredentials([
          file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE'),
          usernamePassword(credentialsId: 'harbor-creds', usernameVariable: 'HARBOR_USER', passwordVariable: 'HARBOR_PASS')
        ]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            # Ensure namespace
            kubectl get ns ${NAMESPACE} >/dev/null 2>&1 || kubectl create ns ${NAMESPACE}

            # Create/Update regcred in target namespace
            kubectl -n ${NAMESPACE} create secret docker-registry regcred \
              --docker-server=${REGISTRY} \
              --docker-username="$HARBOR_USER" \
              --docker-password="$HARBOR_PASS" \
              --docker-email="ci@example.com" \
              --dry-run=client -o yaml | kubectl apply -f -
          '''
        }
      }
    }

    stage('Apply Base K8s (SC/PV/PVC)') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            # Replace ${ENV} with ${NAMESPACE} in your k8s templates and apply
            for f in $(find k8s -type f \\( -name "*.yaml" -o -name "*.yml" \\)); do
              echo "Applying $f for ENV=${NAMESPACE}"
              ENV=${NAMESPACE} envsubst < "$f" | kubectl apply -f -
            done
          '''
        }
      }
    }

    /* ===========================
       GitOps: bump Helm values, push to GitHub
       =========================== */
    stage('Update Helm Values & Push (GitOps)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'GH_USER', passwordVariable: 'GH_PAT')]) {
          sh '''
            set -e

            # Update image tags in helm values
            sed -i 's|^  tag: ".*"|  tag: "'${IMAGE_TAG}'"|' backend-hc/backendvalues.yaml
            sed -i 's|^  tag: ".*"|  tag: "'${IMAGE_TAG}'"|' frontend-hc/frontendvalues.yaml

            git config user.email "ci@example.com"
            git config user.name "ci-bot"
            git add backend-hc/backendvalues.yaml frontend-hc/frontendvalues.yaml
            git commit -m "CI: set image tag ${IMAGE_TAG} for ${NAMESPACE}" || echo "No changes to commit"

            git push https://${GH_USER}:${GH_PAT}@github.com/ThanujaRatakonda/kp_9.git HEAD:master
          '''
        }
      }
    }

    /* ===========================
       Argo CD: apply Applications with ENV=${NAMESPACE}
       =========================== */
    stage('Apply Argo CD Applications') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -e
            export KUBECONFIG="$KUBECONFIG_FILE"

            for f in $(find argocd -maxdepth 1 -type f \\( -name "*.yaml" -o -name "*.yml" \\)); do
              echo "Applying Argo CD app: $f with ENV=${NAMESPACE}"
              ENV=${NAMESPACE} envsubst < "$f" | kubectl apply -n argocd -f -
            done

            kubectl get applications.argoproj.io -n argocd || true
          '''
        }
      }
    }

    stage('Quick Check') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set +e
            export KUBECONFIG="$KUBECONFIG_FILE"

            echo "Pods in ${NAMESPACE}:"
            kubectl -n ${NAMESPACE} get pods -o wide

            echo "Services in ${NAMESPACE}:"
            kubectl -n ${NAMESPACE} get svc

            echo "Ingresses in ${NAMESPACE}:"
            kubectl -n ${NAMESPACE} get ingress || true
          '''
        }
      }
    }

  } // stages

  post {
    always {
      archiveArtifacts artifacts: 'reports/**/*', onlyIfSuccessful: false, allowEmptyArchive: true
    }
    success {
      echo "✅ Done: Images built, scanned, pushed; Helm values updated; Argo CD syncing to ${params.NAMESPACE}."
    }
    failure {
      echo "❌ Pipeline failed. Check namespace/regcred stage (kubectl -n ${NAMESPACE}), Trivy gates, or Git push."
    }
  }
}
