# Jenkins
Jenkins fastapi run
Below is a production-grade Jenkins Pipeline example using a Declarative Jenkinsfile.
It reflects common real-world practices: CI vs CD separation, security scans, Docker image build/push, and Kubernetes deployment.

Example 1: Typical CI/CD Jenkinsfile (Docker + Kubernetes)
Pipeline Flow (High Level)
CI


Checkout source


Lint & unit tests


SAST & dependency scanning


Build Docker image


Scan Docker image


Push image to registry


CD
7. Deploy to Kubernetes (staging)
8. Smoke tests
9. Manual approval
10. Deploy to production

Jenkinsfile (Declarative)
pipeline {
    agent any

    environment {
        APP_NAME        = "demo-app"
        REGISTRY        = "docker.io/myorg"
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        IMAGE_FULL      = "${REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
        KUBE_CONTEXT    = "prod-cluster"
        DOCKER_CREDS    = credentials('dockerhub-creds')
        KUBECONFIG_CRED = credentials('kubeconfig-prod')
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/myorg/demo-app.git'
            }
        }

        stage('Lint & Unit Tests') {
            steps {
                sh '''
                  make lint
                  make test
                '''
            }
        }

        stage('SAST & Dependency Scan') {
            steps {
                sh '''
                  trivy fs --exit-code 1 --severity HIGH,CRITICAL .
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker build -t ${IMAGE_FULL} .
                '''
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh '''
                  trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_FULL}
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                  echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin
                  docker push ${IMAGE_FULL}
                '''
            }
        }

        stage('Deploy to Staging') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-staging', variable: 'KUBECONFIG')]) {
                    sh '''
                      kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${IMAGE_FULL} \
                        -n staging
                    '''
                }
            }
        }

        stage('Smoke Tests') {
            steps {
                sh '''
                  curl -f http://staging.demo-app.local/health
                '''
            }
        }

        stage('Manual Approval') {
            steps {
                input message: 'Approve deployment to production?'
            }
        }

        stage('Deploy to Production') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG')]) {
                    sh '''
                      kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${IMAGE_FULL} \
                        -n production
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully"
        }
        failure {
            echo "Pipeline failed"
        }
        always {
            cleanWs()
        }
    }
}


Example 2: Minimal Jenkins Pipeline (Interview-Friendly)
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/myorg/demo-app.git'
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t demo-app:latest .'
            }
        }

        stage('Test') {
            steps {
                sh 'make test'
            }
        }

        stage('Deploy') {
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }
}


Key Interview Points You Should Mention


Declarative pipelines are preferred for readability and governance


CI and CD stages should be logically separated


Security is shift-left (SAST, dependency scanning early)


Secrets handled via Jenkins Credentials, never hard-coded


Manual approval before production is common in regulated environments


Jenkins is often combined with:


GitHub Actions / ArgoCD (GitOps)


Vault / External Secrets


Shared Libraries for reuse





If you want, I can:


Convert this to Jenkins Shared Library


Provide GitHub Actions equivalent


Add Helm-based deployment


Add blue/green or canary strategy


Tailor it exactly to DevOps interview answers


Tell me how deep you want to go.
