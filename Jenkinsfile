pipeline {
    agent any

    environment {
        DOCKERHUB_USER   = "csk1234"
        FRONTEND_IMAGE   = "incops-frontend"
        BACKEND_IMAGE    = "incops-backend"
    }

    options {
        timeout(time: 30, unit: 'MINUTES') // prevent jobs hanging forever
    }

    tools {
        git 'Default' // removes "Selected Git installation does not exist" warning
    }

    stages {
        stage("System Cleanup") {
            steps {
                sh '''
                docker system prune -f || true
                cleanWs

                # Ensure a persistent buildx builder exists and uses host networking
                if ! docker buildx inspect jenkins-builder >/dev/null 2>&1; then
                  docker buildx create --name jenkins-builder --use --driver docker-container --driver-opt network=host
                  # Enable QEMU emulation for multi-arch builds (amd64 on ARM hosts)
                  docker run --rm --privileged multiarch/qemu-user-static --reset -p yes || true
                  docker buildx inspect jenkins-builder --bootstrap || true
                else
                  # If builder exists, show its config; if it lacks host networking you can recreate it manually
                  docker buildx inspect jenkins-builder || true
                fi
                '''
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

        stage("Docker Login") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'csk-dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                }
            }
        }

        stage("Build & Push Backend Image") {
            steps {
                dir("backend") {
                    sh '''
                    docker buildx build --builder jenkins-builder --platform linux/amd64,linux/arm64 \
                      -t ${DOCKERHUB_USER}/${BACKEND_IMAGE}:latest --push .
                    '''
                }
            }
        }

        stage("Build & Push Frontend Image") {
            steps {
                dir("frontend") {
                    sh '''
                    docker buildx build --builder jenkins-builder --platform linux/amd64,linux/arm64 \
                      -t ${DOCKERHUB_USER}/${FRONTEND_IMAGE}:latest --push .
                    '''
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
                sh """
                microk8s.kubectl apply -f infra/k8s/
                microk8s.kubectl apply -f backend/k8s/
                microk8s.kubectl apply -f frontend/k8s/
                microk8s.kubectl rollout restart deployment backend
                microk8s.kubectl rollout restart deployment frontend
                microk8s.kubectl rollout status deployment backend --timeout=120s
                microk8s.kubectl rollout status deployment frontend --timeout=120s
                """
            }
        }

        stage("DAST - Ingress Scan") {
            steps {
                sh '''
                mkdir -p reports
                chmod -R 777 reports

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
            sh 'docker logout || true'
        }
    }
}
