pipeline {
    agent any

    environment {
        APP_NAME = 'pbl3-cicd-demo'
        IMAGE_NAME = 'pbl3-cicd-demo:latest'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME} .'
            }
        }

        stage('Load Image into Minikube') {
            steps {
                sh '''
                    set -o pipefail
                    docker save ${IMAGE_NAME} | docker exec -i minikube ctr -n k8s.io images import -
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
                sh 'kubectl apply -f k8s/service.yaml'
            }
        }

        stage('Verify Deployment') {
            steps {
                sh 'kubectl rollout status deployment/pbl3-app --timeout=120s'
                sh 'kubectl get deployments'
                sh 'kubectl get pods -l app=pbl3-app'
                sh 'kubectl get services pbl3-app-service'
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    kubectl port-forward service/pbl3-app-service 18080:80 \
                        > /tmp/pbl3-port-forward.log 2>&1 &
                    PF_PID=$!

                    trap 'kill $PF_PID 2>/dev/null || true' EXIT

                    for i in $(seq 1 15); do
                        if curl -fsS http://127.0.0.1:18080/ > /tmp/pbl3-response.html; then
                            break
                        fi
                        sleep 2
                    done

                    grep -q "Cloud & DevOps Dashboard" /tmp/pbl3-response.html
                    echo "Smoke test passed: application returned the expected page."
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'CI/CD pipeline failed. Check the stage logs.'
        }
    }
}
