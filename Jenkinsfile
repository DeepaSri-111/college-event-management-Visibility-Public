
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
                // Capture only the image name, not the echoed CMD command
                def previousImage = bat(
                    script: '@echo off\r\nkubectl get deployment college-event-management-deployment -o jsonpath="{.spec.template.spec.containers[0].image}"',
                    returnStdout: true
                ).trim()

                echo "Previous working image: ${previousImage}"

                def targetImage
                if (params.RELEASE_MODE == 'FAULTY_TEST') {
                    targetImage = "${IMAGE_NAME}:bad-version"
                    echo "Testing rollback using a deliberately invalid image."
                } else {
                    targetImage = "${IMAGE_NAME}:${BUILD_NUMBER}"
                }

                echo "Attempting deployment with: ${targetImage}"

                bat "kubectl set image deployment/college-event-management-deployment college-event-management=${targetImage}"

                int rolloutStatus = bat(
                    script: '@echo off\r\nkubectl rollout status deployment/college-event-management-deployment --timeout=90s',
                    returnStatus: true
                )

                if (rolloutStatus != 0) {
                    echo "Deployment failed. Restoring the previous working image."

                    bat "kubectl set image deployment/college-event-management-deployment college-event-management=${previousImage}"

                    int rollbackStatus = bat(
                        script: '@echo off\r\nkubectl rollout status deployment/college-event-management-deployment --timeout=180s',
                        returnStatus: true
                    )

                    if (rollbackStatus != 0) {
                        bat 'kubectl get pods -l app=college-event-management'
                        error('Rollback failed. Check Kubernetes pod status.')
                    }

                    bat 'kubectl get deployment college-event-management-deployment'
                    bat 'kubectl get pods -l app=college-event-management'

                    if (params.RELEASE_MODE == 'FAULTY_TEST') {
                        error('Expected faulty deployment detected. Automatic rollback completed successfully.')
                    }

                    error('Deployment failed. Previous image restored.')
                }

                bat 'kubectl scale deployment/college-event-management-deployment --replicas=3'
                bat 'kubectl rollout status deployment/college-event-management-deployment --timeout=180s'
                bat 'kubectl get deployment college-event-management-deployment'
                bat 'kubectl get pods -l app=college-event-management'
                bat 'kubectl get service college-event-management-service'
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
