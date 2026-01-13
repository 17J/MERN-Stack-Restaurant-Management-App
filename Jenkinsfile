pipeline {
    agent any

    tools {
        nodejs "nodejs-23.8"
    }

    environment {
        SCANNER_HOME       = tool 'sonar-scanner'
        CLIENT_IMAGE_NAME  = 'frontend-restaurant-mern'
        SERVER_IMAGE_NAME  = 'backend-restaurant-mern'
        IMAGE_TAG          = "${BUILD_NUMBER}"
        DOCKER_REGISTRY    = '17rj'
        K8S_NAMESPACE      = 'mern-restaurant'          
        RECIPIENTS         = 'demo@demo.com, team@demo.com'
        SLACK_CHANNEL      = '#deployments'             
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/17J/MERN-Stack-Restaurant-Management-App.git'
            }
        }

        stage('Gitleaks Secret Scanning') {
            steps {
                sh '''
                    gitleaks detect --no-git --verbose --redact \
                    --exit-code 1 --report-format json --report-path gitleaks-report.json
                '''
            }
        }

        stage('Install Dependencies & Lint') {
            parallel {
                stage('Frontend') {
                    steps {
                        dir('client') {
                            sh '''
                                npm ci
                                npm run lint || true         
                                npm run build                 
                            '''
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        dir('server') {
                            sh '''
                                npm ci
                                npm run lint || true
                            '''
                        }
                    }
                }
            }
        }

        stage('Snyk SCA Scan') {
            parallel {
                stage('Frontend') {
                    steps {
                        dir('client') {
                            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                                sh '''
                                    snyk auth $SNYK_TOKEN
                                    snyk test --severity-threshold=high \
                                    --json-file-output=snyk-frontend.json || true
                                '''
                            }
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        dir('server') {
                            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                                sh '''
                                    snyk auth $SNYK_TOKEN
                                    snyk test --severity-threshold=high \
                                    --json-file-output=snyk-backend.json || true
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=mern-restaurant-app \
                        -Dsonar.sources=client/src,server/src \
                        -Dsonar.exclusions=**/node_modules/**,**/coverage/**,**/dist/** \
                        -Dsonar.javascript.lcov.reportPaths=client/coverage/lcov.info,server/coverage/lcov.info
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            parallel {
                stage('Frontend FS') {
                    steps {
                        sh 'trivy fs --severity HIGH,CRITICAL --exit-code 1 --format table -o trivy-fs-frontend.html client/'
                    }
                }
                stage('Backend FS') {
                    steps {
                        sh 'trivy fs --severity HIGH,CRITICAL --exit-code 1 --format table -o trivy-fs-backend.html server/'
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            parallel {
                stage('Frontend Docker') {
                    steps {
                        dir('client') {
                            withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                                sh """
                                    docker build --pull -t ${DOCKER_REGISTRY}/${CLIENT_IMAGE_NAME}:${IMAGE_TAG} .
                                    docker push ${DOCKER_REGISTRY}/${CLIENT_IMAGE_NAME}:${IMAGE_TAG}
                                """
                            }
                        }
                    }
                }
                stage('Backend Docker') {
                    steps {
                        dir('server') {
                            withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                                sh """
                                    docker build --pull -t ${DOCKER_REGISTRY}/${SERVER_IMAGE_NAME}:${IMAGE_TAG} .
                                    docker push ${DOCKER_REGISTRY}/${SERVER_IMAGE_NAME}:${IMAGE_TAG}
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            parallel {
                stage('Trivy Image Scan') {
                    steps {
                        sh """
                            trivy image --severity HIGH,CRITICAL --exit-code 1 \
                            --format table -o trivy-frontend-image.html \
                            ${DOCKER_REGISTRY}/${CLIENT_IMAGE_NAME}:${IMAGE_TAG}
                        """
                        sh """
                            trivy image --severity HIGH,CRITICAL --exit-code 1 \
                            --format table -o trivy-backend-image.html \
                            ${DOCKER_REGISTRY}/${SERVER_IMAGE_NAME}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kube-cred']) {
                    sh '''
                        set -e
                        kubectl apply -f k8s/backend-deployment-service.yml -n ${K8S_NAMESPACE}
                        kubectl apply -f k8s/db-ds.yml -n ${K8S_NAMESPACE}
                        kubectl apply -f k8s/frontend-deployment-service.yml -n ${K8S_NAMESPACE}
                        kubectl apply -f k8s/frontend-service.yaml -n ${K8S_NAMESPACE}
                        kubectl apply -f k8s/ingress.yml -n ${K8S_NAMESPACE}
                        kubectl apply -f k8s/secrets-configmap.yml -n ${K8S_NAMESPACE} || true

                        # Wait for rollout
                        kubectl rollout status deployment/backend -n ${K8S_NAMESPACE} --timeout=120s
                        kubectl rollout status deployment/frontend -n ${K8S_NAMESPACE} --timeout=120s
                    '''
                }
            }
        }

        stage('Retrieve Application URL') {
            steps {
                withKubeConfig([credentialsId: 'kube-cred']) {
                    script {
                        env.APP_URL = sh(
                            script: "kubectl get ingress frontend-ingress -n ${K8S_NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                            returnStdout: true
                        ).trim() ?: sh(
                            script: "kubectl get svc frontend-service -n ${K8S_NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                            returnStdout: true
                        ).trim()

                        if (!env.APP_URL) {
                            error "Application URL not found. Check ingress/service."
                        }
                    }
                }
                echo "App URL: https://${env.APP_URL}"
            }
        }

        stage('Kubernetes Verify') {
            steps {
                withKubeConfig([credentialsId: 'kube-cred']) {
                    sh """
                        kubectl get pods -n ${K8S_NAMESPACE}
                        kubectl get svc -n ${K8S_NAMESPACE}
                        kubectl get ingress -n ${K8S_NAMESPACE}
                    """
                }
            }
        }


        stage('OWASP ZAP DAST Scan') {
            steps {
                sh """
                    docker run --rm -v \$(pwd):/zap/wrk/:rw \
                    -t ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py -t https://${APP_URL} \
                    -r zap-report.html -I
                """
                archiveArtifacts 'zap-report.html', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/*.{json,html}', allowEmptyArchive: true
        }
        success {
            emailext(
                subject: "✅ SUCCESS: MERN Restaurant Build #${BUILD_NUMBER}",
                body: """
                <h2>Build Successful!</h2>
                <p>Job: ${JOB_NAME}</p>
                <p>Build: #${BUILD_NUMBER}</p>
                <p>Image Tag: ${IMAGE_TAG}</p>
                <p>App URL: <a href="https://${APP_URL}">https://${APP_URL}</a></p>
                <p>Console: <a href="${BUILD_URL}console">here</a></p>
                """,
                to: "${RECIPIENTS}",
                mimeType: "text/html"
            )
            slackSend channel: SLACK_CHANNEL, color: 'good', message: "✅ MERN Restaurant deployed! Tag: ${IMAGE_TAG} URL: https://${APP_URL}"
        }
        failure {
            emailext(
                subject: "❌ FAILURE: MERN Restaurant Build #${BUILD_NUMBER}",
                body: "Build failed! Check console: ${BUILD_URL}console",
                to: "${RECIPIENTS}",
                mimeType: "text/html"
            )
            slackSend channel: SLACK_CHANNEL, color: 'danger', message: "❌ MERN Restaurant pipeline failed! Build #${BUILD_NUMBER}"
        }
    }
}