pipeline {
    agent any

    environment {
        MONGO_URI = credentials('mongo-uri')
        MONGO_USERNAME = credentials('mongo-username')
        MONGO_PASSWORD = credentials('mongo-password')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                always {
                    junit 'test-results.xml'
                    cobertura coberturaReportFile: 'coverage/cobertura-coverage.xml'
                }
            }
        }

        stage('Publish') {
            when {
                branch 'main'
            }
            steps {
                sh 'docker build -t solar-system:latest .'
                sh 'docker push solar-system:latest'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}