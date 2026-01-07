# Jenkins
Jenkins fastapi run
Below is a production-grade Jenkins Pipeline example using a Declarative Jenkinsfile.
It reflects common real-world practices: CI vs CD separation, security scans, Docker image build/push, and Kubernetes deployment.

Example 1: Typical CI/CD Jenkinsfile (Docker + Kubernetes)
agent (any, label 'docker'),environment (setup environment envirables 


Pipeline Flow (High Level)
CI
1)pick agent 
Jenkins selects an available worker (node)

Workspace is created

All stages run on this agent unless overridden

Production note

Often replaced with:

agent { label 'docker' }

agent { kubernetes { ... } }

2)Checkout source


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

------------------------------------------------------------------------------------------------------------------------------------------------------------
summary and details explanation
agent:label:Linux,docker for agent with docker)→

environment :setting environment variables for the pipeline :app name, image_tag=env.build number,creadentials to docker registry,to kubernetes etc.

stage: step: code checkout : git branch main ,will clone the main repo

stage:step : linting and automated tests: make test

stage: static code analysis and dependency scanning (sonarqube ,snik)

stage: step :buid the docker image -sh 'docker build -t app:${BUILD_NUMBER} .’

stage:step: scan the image ,with tools like trivy: 

trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_FULL}

stage: push the image:docker push ${IMAGE_FULL}

stage('Deploy to Staging') kubectl apply    -n staging

---

-cd part 

stage('Smoke Tests') curl -f http://staging.demo-app.local/health 

stage('Manual Approval') { input message: 'Approve deployment to production?’ use need to press yes ,before the pipeline will move to production stage 

stage('Deploy to Production') kubectl  apply -n production


detailed explanation
### Pipeline Structure Overview

This meeting covered the complete flow of a Jenkins pipeline, from initial configuration through deployment to production.  

### Agent Configuration

- The first step in a Jenkins pipeline is defining the **agent**, which specifies where the pipeline will run
- **Labels** are used to group agents with similar capabilities (e.g., `Linux` for Linux machines, `Docker` for agents with Docker installed)
- In production environments, specific labels should be used instead of `agent any` to ensure pipelines run on the right set of nodes and maintain consistency

### Environment Variables

- After defining the agent, environment variables are set as key-value pairs for configuration
- Common examples include:
    - `JAVA_HOME` to specify Java installation path
    - `BUILD_NUMBER` to track the current build number
- `$BUILD_NUMBER` is a built-in Jenkins environment variable that represents the unique number of the current build, incremented with each run
- Environment variables are set according to each pipeline's specific needs

### Pipeline Stages

### Checkout Stage

- The first stage is typically the **checkout stage**, where source code is pulled from the repository
- Each pipeline has a **workspace** - a directory on the agent where Jenkins places all files for that pipeline run, including checked-out code and generated artifacts
- The `checkout` step in Jenkins automatically handles cloning; the branch is specified and Jenkins performs the clone in the background
- No need to run `git clone` manually

### Linting and Unit Tests Stage

- After checkout, the next stage typically includes linting (checking code for syntax errors or style issues) and unit testing
- Example commands: `make lint` and `make test`
- `make` is a general-purpose build automation tool that can run any defined tasks (linting, testing, deployment), not just compilation
- A **Makefile** should be included in the Git repository to define these commands and automate tasks consistently
- This approach can be used with many programming languages: C, Python, Go, Java, etc.

### Static Code Analysis and Dependency Scanning

- This stage helps catch potential security vulnerabilities, code smells, and outdated dependencies
- **Tools and syntax**:
    - Static code analysis: **SonarCube** (command: `sonar-scanner`)
    - Dependency scanning: **OWASP Dependency Check** (command: `dependency-check --scan .`) or **Snyk** (command: `snyk test`)

### Build/Packaging Stage

- After analysis and scanning, the code is compiled and the final artifact is created (binary, deployable package, or Docker image)
- Docker images can be built using `docker build`

### Image Scanning Stage

- Tools like **Trivy** or **Anchor** are used to analyze Docker images for vulnerabilities before pushing to a registry

### Push to Docker Registry

- The built and scanned image is pushed to a Docker registry (Docker Hub or private registry) using `docker push`
- This stage marks the **end of the CI stages**

### Deploy to Staging

- After CI stages, the pipeline moves into CD stages
- Deployment to staging uses tools like Kubernetes, Helm, or direct Docker commands
- Example kubectl command: `kubectl apply -f .yaml` to apply configuration and deploy to the staging environment

### Integration/E2E Testing

- After deploying to staging, integration or end-to-end tests are run to ensure everything works as expected in that environment

### Manual Approval

- After tests pass, a manual approval step occurs before promoting to production
- Jenkins uses an **input step** for manual approval, which pauses the pipeline and prompts a person to confirm before proceeding
- Someone presses "yes" or "approve" in the Jenkins interface to continue

### Production Deployment

- After manual approval, the pipeline continues with production deployment

### Post Section

- The **post section** defines actions that run after the pipeline completes (whether it succeeds or fails)
- Common post actions include:
    - Sending notifications
    - Archiving artifacts





Tell me how deep you want to go.
