pipeline {
    agent any
    
    triggers {
        // Auto-trigger saat ada push dari GitHub via webhook
        githubPush()
    }
    
    tools {
        maven 'Maven 3.8.7' // Pastikan nama ini sesuai dengan konfigurasi Maven di Jenkins Global Tool Configuration
        jdk 'JDK-21' // Pastikan nama ini sesuai dengan konfigurasi JDK di Jenkins Global Tool Configuration
    }
    
    environment {
        // Docker Registry Configuration
        DOCKER_REGISTRY = '' // Kosongkan untuk Docker Hub
        DOCKER_IMAGE_NAME = 'h4lfbaked/springboot-test-boost'
        DOCKER_CREDENTIALS_ID = 'docker-password' // ID credentials di Jenkins
        
        // Application Configuration
        APP_NAME = 'springboot-test-boost'
        APP_VERSION = "1.0.0"
    }
    
    stages {
        stage('1. Checkout') {
            steps {
                echo '=== Stage 1: Checking out code from repository ==='
                checkout scm
                script {
                    env.GIT_COMMIT_MSG = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    env.GIT_AUTHOR = sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim()
                    echo "Commit: ${env.GIT_COMMIT_MSG}"
                    echo "Author: ${env.GIT_AUTHOR}"
                }
            }
        }
        
        stage('2. Build & Test') {
            steps {
                echo '=== Stage 2: Building and Testing Application ==='
                sh 'mvn clean test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('3. Package') {
            steps {
                echo '=== Stage 3: Packaging Application ==='
                sh 'mvn package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                    echo "JAR artifact created successfully"
                }
            }
        }
        
        stage('4. Install to Maven Repository') {
            steps {
                echo '=== Stage 4: Installing JAR to Maven Repository ==='
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo "JAR installed to Maven repository successfully"
                    echo "You can now use this library as a dependency in other projects"
                }
            }
        }
        
        stage('5. Docker Build & Push') {
            steps {
                echo '=== Stage 5: Building and Pushing Docker Image ==='
                script {
                    def dockerImage = docker.build("${DOCKER_IMAGE_NAME}:${APP_VERSION}")
                    
                    docker.withRegistry('', "${DOCKER_CREDENTIALS_ID}") {
                        dockerImage.push("${APP_VERSION}")
                        dockerImage.push('latest')
                    }
                    
                    echo "Docker image pushed: ${DOCKER_IMAGE_NAME}:${APP_VERSION}"
                }
            }
        }
        
        stage('6. Deploy') {
            steps {
                echo '=== Stage 6: Deploying Application ==='
                script {
                    def port = '8079'
                    def envSuffix = 'prod'
                    
                    if (env.BRANCH_NAME == 'dev') {
                        port = '8078'
                        envSuffix = 'dev'
                        echo "Deploying to Development environment..."
                    } else if (env.BRANCH_NAME == 'staging') {
                        port = '8077'
                        envSuffix = 'uat'
                        echo "Deploying to Staging environment..."
                    } else if (env.BRANCH_NAME == 'main') {
                        echo "Deploying to Production environment..."
                        input message: 'Deploy to Production?', ok: 'Deploy'
                    }
                    
                    sh """
                        docker stop ${APP_NAME}-${envSuffix} || true
                        docker rm ${APP_NAME}-${envSuffix} || true
                    """
                    
                    sh """
                        docker run -d \
                        --name ${APP_NAME}-${envSuffix} \
                        -p ${port}:8080 \
                        --restart unless-stopped \
                        ${DOCKER_IMAGE_NAME}:${APP_VERSION}
                    """
                    
                    echo "Waiting for application to start..."
                    sleep(time: 30, unit: 'SECONDS')
                    
                    sh """
                        curl -f http://localhost:${port}/api/health || exit 1
                    """
                    
                    echo "Application deployed successfully on port ${port}"
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
            echo "Application Version: ${APP_VERSION}"
            echo "Docker Image: ${DOCKER_IMAGE_NAME}:${APP_VERSION}"
        }
        
        failure {
            echo 'Pipeline failed!'
            echo "Please check the logs for details."
        }
        
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
            sh 'docker image prune -f || true'
        }
    }
}