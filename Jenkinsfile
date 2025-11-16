pipeline {
    agent any
    
    parameters {
        string(name: 'DEPLOY_ENV', defaultValue: 'staging', description: 'Deployment environment: staging or production')
    }
    
    environment {
        REPO_URL = 'https://github.com/SabbirIrfan/solar-system-gitea-pipe-pilot.git'
        NODE_ENV = 'production'
        APP_NAME = 'solar-system'
        IMAGE_NAME = 'solar-system-app'
        DOCKER_REGISTRY = 'your-docker-registry.example.com'
        MONGO_URI = credentials('mongo-uri')    // Jenkins Credential IDs for sensitive data
        MONGO_USERNAME = credentials('mongo-username')
        MONGO_PASSWORD = credentials('mongo-password')
        NOTIFY_EMAIL = 'team@example.com'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        ansiColor('xterm')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        skipDefaultCheckout true
    }

    stages {
        stage('Checkout') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "refs/heads/${env.BRANCH_NAME ?: 'main'}"]],
                        userRemoteConfigs: [[url: env.REPO_URL]]
                    ])
                }
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine3.17'
                    args '-u root:root'
                    reuseNode true
                }
            }
            environment {
                NODE_ENV = 'production'
            }
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        sh 'npm ci'
                        sh 'npm run build || echo "No build script defined, skipping"'
                    }
                }
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine3.17'
                    args '-u root:root'
                    reuseNode true
                }
            }
            environment {
                NODE_ENV = 'test'
            }
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        sh 'npm test'
                    }
                }
            }
            post {
                always {
                    junit '**/test-results.xml' // Assuming mocha-junit-reporter or similar outputs test results 
                    archiveArtifacts artifacts: 'coverage/**', allowEmptyArchive: true
                }
                failure {
                    script {
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }

        stage('Analysis') {
            agent {
                docker {
                    image 'node:18-alpine3.17'
                    args '-u root:root'
                    reuseNode true
                }
            }
            steps {
                script {
                    // Placeholder: Insert static code analysis or linting logic
                    timeout(time: 5, unit: 'MINUTES') {
                        sh 'npm run lint || echo "No lint script defined, skipping"'
                    }
                }
            }
            post {
                failure {
                    script {
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        sh """
                            docker build --pull -t ${env.DOCKER_REGISTRY}/${env.IMAGE_NAME}:${env.BRANCH_NAME ?: 'latest'} .
                        """
                    }
                }
            }
        }

        stage('Push Docker Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                    branch 'release/**'
                }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-registry-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        timeout(time: 5, unit: 'MINUTES') {
                            sh """
                                echo "$DOCKER_PASS" | docker login ${env.DOCKER_REGISTRY} -u "$DOCKER_USER" --password-stdin
                                docker push ${env.DOCKER_REGISTRY}/${env.IMAGE_NAME}:${env.BRANCH_NAME ?: 'latest'}
                                docker logout ${env.DOCKER_REGISTRY}
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                }
            }
            steps {
                script {
                    timeout(time: 15, unit: 'MINUTES') {
                        // Example deployment step: Could call kubectl, helm, ssh, ansible, etc.
                        echo "Deploying ${env.DOCKER_REGISTRY}/${env.IMAGE_NAME}:${env.BRANCH_NAME ?: 'latest'} to environment ${params.DEPLOY_ENV}"
                        // Add real deployment commands here
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                emailext (
                    to: "${env.NOTIFY_EMAIL}",
                    subject: "SUCCESS: Build ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """Build completed successfully.
                    Job: ${env.JOB_NAME}
                    Build Number: ${env.BUILD_NUMBER}
                    Branch: ${env.BRANCH_NAME ?: 'main'}
                    URL: ${env.BUILD_URL}"""
                )
            }
        }
        failure {
            script {
                emailext (
                    to: "${env.NOTIFY_EMAIL}",
                    subject: "FAILURE: Build ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """Build failed.
                    Job: ${env.JOB_NAME}
                    Build Number: ${env.BUILD_NUMBER}
                    Branch: ${env.BRANCH_NAME ?: 'main'}
                    URL: ${env.BUILD_URL}"""
                )
            }
        }
        cleanup {
            script {
                echo 'Cleaning up workspace and docker images...'
                deleteDir()

                // Remove dangling docker images created by builds (optional)
                sh script: '''
                    docker image prune -f || true
                ''', returnStatus: true
            }
        }
    }
}