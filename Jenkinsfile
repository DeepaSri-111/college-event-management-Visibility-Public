
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'deepasri111/college-event-management'
        DEPLOYMENT_NAME = 'college-event-management-deployment'
        CONTAINER_NAME = 'college-event-management'
        SERVICE_NAME = 'college-event-management-service'
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Test Application') {
            steps {
                bat 'findstr /C:"CampusConnect" index.html'
                bat 'findstr /C:"registrationForm" index.html'
                bat 'findstr /C:"facultySummary" index.html'
                bat 'findstr /C:"CodeSprint Hackathon" index.html'
                echo 'Application checks passed.'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = "${IMAGE_NAME}:${BUILD_NUMBER}"
                }
                bat "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    bat 'docker login -u "%DOCKER_USER%" -p "%DOCKER_PASS%"'
                    bat "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                    bat 'docker logout'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig-file',
                    variable: 'KUBECONFIG'
                )]) {
                    bat 'kubectl get nodes'
                    bat "kubectl set image deployment/${DEPLOYMENT_NAME} ${CONTAINER_NAME}=${IMAGE_NAME}:${BUILD_NUMBER}"
                    bat "kubectl rollout status deployment/${DEPLOYMENT_NAME} --timeout=180s"
                    bat "kubectl scale deployment/${DEPLOYMENT_NAME} --replicas=3"
                    bat "kubectl get pods -l app=college-event-management"
                    bat "kubectl get service ${SERVICE_NAME}"
                }
            }
        }
    }

    post {
        success {
            echo 'College Event Management deployment completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }
    }
}
