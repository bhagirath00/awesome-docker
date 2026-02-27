# Docker with Jenkins

## Introduction

Jenkins remains the dominant CI/CD platform in enterprise environments. Its flexibility, plugin ecosystem, and self-hosted nature make it the go-to choice for organizations with complex on-premises infrastructure or strict data sovereignty requirements.

This file covers implementing a complete Docker build-test-scan-push-deploy pipeline in Jenkins using both Declarative and Scripted Pipeline syntax, with AWS ECR integration.

---

## Jenkins Docker Prerequisites

Jenkins needs Docker access to build and run containers. There are two approaches:

### Option 1: Docker on the Jenkins Host (Recommended)

Install Docker on the Jenkins server. Run the Jenkins agent as a user with access to the Docker socket:

```bash
# Add the jenkins user to the docker group
sudo usermod -aG docker jenkins

# Restart Jenkins
sudo systemctl restart jenkins
```

Jenkins can now run `docker build`, `docker run`, and `docker push` commands directly.

### Option 2: Docker-in-Docker (DinD)

Mount the host's Docker socket into the Jenkins agent container:

```yaml
# docker-compose.yml for Jenkins with Docker access
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock   # Docker socket mount
    ports:
      - "8080:8080"
      - "50000:50000"
```

The Jenkins container now has access to the host's Docker daemon.

---

## Declarative Jenkinsfile — Basic Docker Pipeline

```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any   // Run on any available Jenkins agent

    environment {
        AWS_REGION     = 'us-east-1'
        ECR_REGISTRY   = '123456789012.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY = 'myapp'
        IMAGE_TAG      = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${GIT_COMMIT.take(8)}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Lint Dockerfile') {
            steps {
                sh 'docker run --rm -i hadolint/hadolint < Dockerfile'
            }
        }

        stage('Build Image') {
            steps {
                sh """
                    docker build \
                      --build-arg GIT_COMMIT=${GIT_COMMIT.take(8)} \
                      --build-arg BUILD_DATE=\$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                      -t ${IMAGE_TAG} \
                      -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest \
                      .
                """
            }
        }

        stage('Run Tests') {
            steps {
                sh """
                    docker run --rm \
                      -v \$(pwd)/test-results:/results \
                      ${IMAGE_TAG} \
                      pytest tests/ -v --junitxml=/results/test-results.xml
                """
            }
            post {
                always {
                    junit 'test-results/test-results.xml'
                }
            }
        }

        stage('Security Scan') {
            steps {
                sh """
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy image \
                      --exit-code 1 \
                      --severity CRITICAL,HIGH \
                      ${IMAGE_TAG}
                """
            }
        }

        stage('Push to ECR') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                          docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker push ${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                    """
                }
            }
        }

        stage('Deploy to ECS') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh """
                        aws ecs update-service \
                          --region ${AWS_REGION} \
                          --cluster production \
                          --service myapp-api \
                          --force-new-deployment

                        aws ecs wait services-stable \
                          --region ${AWS_REGION} \
                          --cluster production \
                          --services myapp-api
                    """
                }
            }
        }
    }

    post {
        always {
            // Clean up Docker resources to prevent disk exhaustion
            sh """
                docker rmi ${IMAGE_TAG} || true
                docker rmi ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest || true
                docker system prune -f
            """
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ Deployed myapp:${GIT_COMMIT.take(8)} to production"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ Pipeline failed for myapp:${GIT_COMMIT.take(8)}"
            )
        }
    }
}
```

---

## Using the Jenkins Docker Pipeline Plugin

The [Docker Pipeline Plugin](https://plugins.jenkins.io/docker-workflow/) provides native Docker DSL for Jenkinsfiles, making container operations cleaner:

```groovy
pipeline {
    agent any

    environment {
        ECR_REGISTRY   = '123456789012.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY = 'myapp'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    // Build and capture the image object
                    def image = docker.build("${ECR_REGISTRY}/${ECR_REPOSITORY}:${GIT_COMMIT.take(8)}", \
                        "--build-arg GIT_COMMIT=${GIT_COMMIT.take(8)} .")

                    // Run tests inside the built image
                    image.inside {
                        sh 'pytest tests/ -v'
                    }

                    // Push to ECR
                    docker.withRegistry("https://${ECR_REGISTRY}", 'ecr:us-east-1:aws-ecr-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
    }
}
```

The `docker.withRegistry` call handles ECR authentication using the AWS ECR Credentials Plugin.

---

## Running Pipeline Stages in Docker Containers

Each stage can run in a specific Docker container as the build environment:

```groovy
pipeline {
    agent none   // No default agent — each stage specifies its own

    stages {
        stage('Lint') {
            agent {
                docker {
                    image 'hadolint/hadolint:latest-alpine'
                    reuseNode true
                }
            }
            steps {
                sh 'hadolint Dockerfile'
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'python:3.11-slim'
                    args '-v /tmp:/tmp'   // Mount tmp for test artifacts
                }
            }
            steps {
                sh 'pip install -r requirements-test.txt -q'
                sh 'pytest tests/ -v --junitxml=/tmp/test-results.xml'
                sh 'cp /tmp/test-results.xml test-results.xml'
            }
            post {
                always { junit 'test-results.xml' }
            }
        }

        stage('Security Scan') {
            agent {
                docker {
                    image 'aquasec/trivy:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:latest'
            }
        }
    }
}
```

---

## Parallel Stages for Faster Pipelines

Run independent stages in parallel to reduce total pipeline time:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'docker build -t myapp:${GIT_COMMIT.take(8)} .'
            }
        }

        // Run lint, unit tests, and security scan in parallel
        stage('Quality Gate') {
            parallel {
                stage('Dockerfile Lint') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint < Dockerfile'
                    }
                }

                stage('Unit Tests') {
                    steps {
                        sh 'docker run --rm myapp:${GIT_COMMIT.take(8)} pytest tests/unit/'
                    }
                }

                stage('Integration Tests') {
                    steps {
                        sh 'docker compose -f docker-compose.test.yml up --abort-on-container-exit'
                    }
                    post {
                        always {
                            sh 'docker compose -f docker-compose.test.yml down'
                        }
                    }
                }
            }
        }

        stage('Push') {
            when { branch 'main' }
            steps {
                sh 'docker push myapp:${GIT_COMMIT.take(8)}'
            }
        }
    }
}
```

---

## Managing Build Caching

Jenkins agents are persistent VMs — build cache accumulates on disk. Manage it:

```groovy
stage('Build') {
    steps {
        sh """
            # Use previous image as cache source
            docker pull ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest || true

            docker build \
              --cache-from ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest \
              -t ${IMAGE_TAG} .
        """
    }
}

// Periodic cleanup job (schedule via Jenkins Cron)
stage('Cleanup') {
    when { triggeredBy 'TimerTrigger' }
    steps {
        sh 'docker system prune -af --volumes'
    }
}
```

For agent disk management, add a cleanup step in `post { always }`:

```groovy
post {
    always {
        sh 'docker image prune -f'
        cleanWs()   // Cleans the Jenkins workspace
    }
}
```

---

## Storing Secrets in Jenkins Credentials

Never put AWS credentials, tokens, or passwords in the Jenkinsfile. Store them in Jenkins Credentials Manager:

```
Jenkins Dashboard → Manage Jenkins → Credentials → Add Credentials
```

Access in pipelines:

```groovy
// Username/Password credential
withCredentials([usernamePassword(
    credentialsId: 'aws-ecr-credentials',
    usernameVariable: 'AWS_ACCESS_KEY_ID',
    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
)]) {
    sh 'aws ecr get-login-password ...'
}

// Secret text
withCredentials([string(
    credentialsId: 'slack-webhook-url',
    variable: 'SLACK_URL'
)]) {
    sh "curl -X POST ${SLACK_URL} -d '{\"text\":\"Deployed!\"}'"
}

// SSH key
withCredentials([sshUserPrivateKey(
    credentialsId: 'deploy-ssh-key',
    keyFileVariable: 'SSH_KEY'
)]) {
    sh "ssh -i ${SSH_KEY} deploy@production-host 'docker compose pull && docker compose up -d'"
}
```

---

## Summary

- Jenkins agents need Docker access — add the `jenkins` user to the `docker` group, or mount the Docker socket into the Jenkins container
- Declarative Pipelines (`pipeline { }`) are the modern Jenkins standard — use them over Scripted Pipelines for clarity and built-in post-step handling
- The Docker Pipeline Plugin provides native `docker.build()`, `image.inside()`, and `docker.withRegistry()` DSL for cleaner Docker operations
- Parallel stages (`parallel { }`) reduce pipeline time by running independent quality checks (lint, unit tests, security scan) simultaneously
- Use `withCredentials()` blocks to inject AWS keys, tokens, and SSH keys from Jenkins Credentials Manager — never hardcode credentials in Jenkinsfiles
- Always clean up in the `post { always }` block — Docker build artifacts, test containers, and dangling images will fill the agent disk if not managed

In the next file, we cover the complete Docker build-push pipeline from first principles — a reusable shell script approach that works across any CI platform.
