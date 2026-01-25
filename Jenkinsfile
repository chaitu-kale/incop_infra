pipeline {
    agent any

    tools {
        git 'Default'
    }

    environment {
        DOCKERHUB_USER = "csk1234"
        FRONTEND_IMAGE = "incops-frontend"
        BACKEND_IMAGE  = "incops-backend"
        KUBECONFIG     = "/var/lib/jenkins/.kube/config"
        PATH = "/opt/codeql/codeql:${env.PATH}"
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

        stage("SAST with CodeQL") {
            steps {
                sh """
                # Backend analysis
                codeql database create backend-db --language=javascript --source-root=backend || true
                codeql database analyze backend-db javascript-security-and-quality.qls --format=sarif --output backend-report.sarif || true

                # Frontend analysis
                codeql database create frontend-db --language=javascript --source-root=frontend || true
                codeql database analyze frontend-db javascript-security-and-quality.qls --format=sarif --output frontend-report.sarif || true
                """
                archiveArtifacts artifacts: '**/*.sarif', fingerprint: true
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
                # Scan backend image
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${BACKEND_IMAGE}:latest > trivy-backend-report.txt || true

                # Scan frontend image
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${FRONTEND_IMAGE}:latest > trivy-frontend-report.txt || true
                """
                archiveArtifacts artifacts: '**/trivy-*.txt', fingerprint: true
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
    }

    post {
        always {
            sh "docker rmi ${DOCKERHUB_USER}/${BACKEND_IMAGE}:latest || true"
            sh "docker rmi ${DOCKERHUB_USER}/${FRONTEND_IMAGE}:latest || true"
        }
    }
}
