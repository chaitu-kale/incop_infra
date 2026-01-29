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

        stage("DAST - Local Container Scan") {
            steps {
                sh '''
                mkdir -p reports
                chmod -R 777 reports

                # Frontend local scan
                docker rm -f test-frontend || true
                docker run -d --name test-frontend -p 8081:3000 ${DOCKERHUB_USER}/${FRONTEND_IMAGE}:latest
                timeout 60 sh -c 'until curl -s http://127.0.0.1:8081 > /dev/null; do sleep 5; done'
                docker run --rm --network="host" \
                  -v $(pwd)/reports:/zap/wrk \
                  zaproxy/zap-stable zap-baseline.py \
                  -t http://127.0.0.1:8081 -r zap-local-frontend.html -I || true
                docker rm -f test-frontend || true

                # Backend local scan
                docker rm -f test-backend || true
                docker run -d --name test-backend -p 8082:5000 ${DOCKERHUB_USER}/${BACKEND_IMAGE}:latest
                timeout 60 sh -c 'until curl -s http://127.0.0.1:8082 > /dev/null; do sleep 5; done'
                docker run --rm --network="host" \
                  -v $(pwd)/reports:/zap/wrk \
                  zaproxy/zap-stable zap-baseline.py \
                  -t http://127.0.0.1:8082 -r zap-local-backend.html -I || true
                docker rm -f test-backend || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/zap-local-*.html', fingerprint: true
                }
            }
        }

        stage("Deploy to K8s") {
            steps {
                sh '''
                kubectl apply -f infra/k8s/
                kubectl apply -f backend/k8s/
                kubectl apply -f frontend/k8s/
                kubectl rollout restart deployment backend
                kubectl rollout restart deployment frontend
                '''
            }
        }

        stage("DAST - Ingress Scan") {
            steps {
                sh '''
                mkdir -p reports
               #chmod -R 777 reports

                docker run --rm -v $(pwd)/reports:/zap/wrk/:rw \
                  zaproxy/zap-stable zap-baseline.py \
                  -t http://myapp.local/frontend \
                  -r zap-frontend-report.html || true

                docker run --rm -v $(pwd)/reports:/zap/wrk/:rw \
                  zaproxy/zap-stable zap-baseline.py \
                  -t http://myapp.local/backend \
                  -r zap-backend-report.html || true
                '''
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
            sh "docker system prune -f || true"
        }
    }
}
