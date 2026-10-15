
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
                echo 'Application tests passed.'
            }
        }

        stage('Build Docker Image') {
            steps {
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

        stage('Deploy with Automatic Rollback') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig-file',
                    variable: 'KUBECONFIG'
                )]) {
                    script {
                        bat "kubectl rollout history deployment/${DEPLOYMENT_NAME}"

                        def previousImage = bat(
                            script: "kubectl get deployment ${DEPLOYMENT_NAME} -o jsonpath=\"{.spec.template.spec.containers[0].image}\"",
                            returnStdout: true
                        ).trim()

                        def targetImage = "${IMAGE_NAME}:${BUILD_NUMBER}"

                        bat "kubectl set image deployment/${DEPLOYMENT_NAME} ${CONTAINER_NAME}=${targetImage}"

                        def rolloutResult = bat(
                            script: "kubectl rollout status deployment/${DEPLOYMENT_NAME} --timeout=120s",
                            returnStatus: true
                        )

                        if (rolloutResult != 0) {
                            echo 'Deployment failed. Rolling back to the previous working image.'

                            bat "kubectl set image deployment/${DEPLOYMENT_NAME} ${CONTAINER_NAME}=${previousImage}"
                            bat "kubectl rollout status deployment/${DEPLOYMENT_NAME} --timeout=180s"

                            error('New deployment failed. Automatic rollback was performed.')
                        }

                        bat "kubectl scale deployment/${DEPLOYMENT_NAME} --replicas=3"
                        bat "kubectl get pods -l app=college-event-management"
                        bat "kubectl get service ${SERVICE_NAME}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the console output and rollback status.'
        }
    }
}
