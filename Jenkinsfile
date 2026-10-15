
pipeline {
    agent any

    parameters {
        choice(
            name: 'RELEASE_MODE',
            choices: ['NORMAL', 'FAULTY_TEST'],
            description: 'NORMAL deploys a valid image. FAULTY_TEST verifies automatic rollback.'
        )
    }

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
                        def previousImage = bat(
                            script: "kubectl get deployment ${DEPLOYMENT_NAME} -o jsonpath=\"{.spec.template.spec.containers[0].image}\"",
                            returnStdout: true
                        ).trim()

                        def targetImage = "${IMAGE_NAME}:${BUILD_NUMBER}"

                        if (params.RELEASE_MODE == 'FAULTY_TEST') {
                            targetImage = "${IMAGE_NAME}:bad-version"
                            echo 'FAULTY_TEST selected: deploying a deliberately nonexistent image tag.'
                        }

                        echo "Previous working image: ${previousImage}"
                        echo "Attempting deployment with: ${targetImage}"

                        bat "kubectl set image deployment/${DEPLOYMENT_NAME} ${CONTAINER_NAME}=${targetImage}"

                        def rolloutResult = bat(
                            script: "kubectl rollout status deployment/${DEPLOYMENT_NAME} --timeout=90s",
                            returnStatus: true
                        )

                        if (rolloutResult != 0) {
                            echo 'Deployment failed or timed out. Restoring the previous working image.'

                            bat "kubectl set image deployment/${DEPLOYMENT_NAME} ${CONTAINER_NAME}=${previousImage}"
                            bat "kubectl rollout status deployment/${DEPLOYMENT_NAME} --timeout=180s"

                            bat "kubectl get deployment ${DEPLOYMENT_NAME}"
                            bat "kubectl get pods -l app=college-event-management"

                            if (params.RELEASE_MODE == 'FAULTY_TEST') {
                                error('Expected faulty deployment detected. Automatic rollback completed successfully.')
                            } else {
                                error('Deployment failed. Previous image restored.')
                            }
                        }

                        bat "kubectl scale deployment/${DEPLOYMENT_NAME} --replicas=3"
                        bat "kubectl rollout status deployment/${DEPLOYMENT_NAME} --timeout=180s"
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
