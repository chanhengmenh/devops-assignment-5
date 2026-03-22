pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonar-token')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=devops-assignment-5 \
                          -Dsonar.projectName="DevOps Assignment 5 - FastAPI" \
                          -Dsonar.projectVersion=1.0 \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=**/__pycache__/**,**/*.pyc,**/venv/**,.git/** \
                          -Dsonar.language=py \
                          -Dsonar.python.version=3 \
                          -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
