pipeline {
    agent any

    tools {
        git 'Default'
    }

    environment {
        DOCKERHUB_USER   = "csk1234"
        FRONTEND_IMAGE   = "incops-frontend"
        BACKEND_IMAGE    = "incops-backend"
        KUBECONFIG       = "/var/lib/jenkins/.kube/config"
    }

    stages {
        stage("System Cleanup") {
            steps {
                echo "Reclaiming disk space before build..."
                sh 'docker system prune -f || true'
                cleanWs()
            }
        }

        stage("Checkout Repos") {
            steps {
                dir("frontend") {
                    git credentialsId: 'chaitu-github-creds', url: 'https://github.com/chaitu-kale/incop_frontend.git', branch: 'main'
                }
                dir("backend") {
                    git credentialsId: 'chaitu-github-creds', url: 'https://github.com/chaitu-kale/incop_backend.git', branch: 'main'
                }
                dir("infra") {
                    git credentialsId: 'chaitu-github-creds', url: 'https://github.com/chaitu-kale/incop_infra.git', branch: 'main'
                }
            }
        }

        stage("Build & Push Backend Image") {
            steps {
                dir("backend") {
                    script {
                        docker.withRegistry('', 'csk-dockerhub-creds') {
                            def img = docker.build("${DOCKERHUB_USER}/${BACKEND_IMAGE}:latest", ".")
                            img.push()
                        }
                    }
                }
            }
        }

        stage("Build & Push Frontend Image") {
            steps {
                dir("frontend") {
                    script {
                        docker.withRegistry('', 'csk-dockerhub-creds') {
                            def img = docker.build("${DOCKERHUB_USER}/${FRONTEND_IMAGE}:latest", ".")
                            img.push()
                        }
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                sh """
                mkdir -p reports
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${BACKEND_IMAGE}:latest > reports/trivy-backend-report.txt || true
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${FRONTEND_IMAGE}:latest > reports/trivy-frontend-report.txt || true
                """
                archiveArtifacts artifacts: 'reports/*.txt', fingerprint: true
            }
        }

        stage("Deploy to K8s") {
            steps {
                sh """
                microk8s.kubectl apply -f infra/k8s/
                microk8s.kubectl apply -f backend/k8s/
                microk8s.kubectl apply -f frontend/k8s/
                microk8s.kubectl rollout restart deployment backend
                microk8s.kubectl rollout restart deployment frontend
                """
            }
        }

        stage("DAST Scan") {
            steps {
                sh """
                mkdir -p reports
                docker run --rm -v $(pwd)/reports:/zap/wrk/:rw \
                  zaproxy/zap-stable zap-baseline.py \
                  -t http://myapp.local/frontend \
                  -r zap-frontend-report.html || true

                docker run --rm -v $(pwd)/reports:/zap/wrk/:rw \
                  zaproxy/zap-stable zap-baseline.py \
                  -t http://myapp.local/backend \
                  -r zap-backend-report.html || true
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/zap-*.html', fingerprint: true
                }
            }
        }
    }

    post {
        always {
            sh "docker rmi ${DOCKERHUB_USER}/${BACKEND_IMAGE}:latest || true"
            sh "docker rmi ${DOCKERHUB_USER}/${FRONTEND_IMAGE}:latest || true"
        }
    }
}
