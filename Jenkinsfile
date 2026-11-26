pipeline {
    agent any

    tools {
        jdk 'JDK'
        maven 'Maven3'
    }

    parameters {
        booleanParam(name: 'DEPLOY_TO_KUBERNETES', defaultValue: false,
                description: 'Push the image and deploy it to the configured Kubernetes cluster')
        string(name: 'IMAGE_REPOSITORY', defaultValue: '',
                description: 'Container image repository, e.g. docker.io/your-user/hotel-booking-system')
        string(name: 'DOCKER_CREDENTIALS_ID', defaultValue: 'dockerhub-creds',
                description: 'Jenkins username/password credential ID for the image registry')
        string(name: 'KUBECONFIG_CREDENTIALS_ID', defaultValue: 'kubeconfig',
                description: 'Jenkins secret-file credential ID containing the Kubernetes config')
    }

    stages {
        stage('Build and verify') {
            steps {
                bat 'mvn -B clean verify'
            }
        }

        stage('Build container') {
            steps {
                script {
                    env.APP_IMAGE = "hotel-booking-system:${env.BUILD_NUMBER}"
                    if (params.DEPLOY_TO_KUBERNETES) {
                        if (!params.IMAGE_REPOSITORY?.trim()
                                || !(params.IMAGE_REPOSITORY ==~ '^[A-Za-z0-9][A-Za-z0-9._/-]*$')) {
                            error('Set IMAGE_REPOSITORY to a valid registry/repository name before deployment.')
                        }
                        env.APP_IMAGE = "${params.IMAGE_REPOSITORY}:${env.BUILD_NUMBER}"
                    }
                }
                bat 'docker build --tag "%APP_IMAGE%" .'
            }
        }

        stage('Push image') {
            when {
                expression { params.DEPLOY_TO_KUBERNETES }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: params.DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'REGISTRY_USERNAME',
                        passwordVariable: 'REGISTRY_PASSWORD')]) {
                    bat '''
                        @echo off
                        echo %REGISTRY_PASSWORD% | docker login --username "%REGISTRY_USERNAME%" --password-stdin
                        if errorlevel 1 exit /b 1
                        docker push "%APP_IMAGE%"
                        if errorlevel 1 exit /b 1
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { params.DEPLOY_TO_KUBERNETES }
            }
            steps {
                withCredentials([file(credentialsId: params.KUBECONFIG_CREDENTIALS_ID,
                        variable: 'KUBECONFIG')]) {
                    bat '''
                        kubectl apply -f k8s/booking-storage.yaml
                        if errorlevel 1 exit /b 1
                        kubectl apply -f k8s/deployment.yaml
                        if errorlevel 1 exit /b 1
                        kubectl apply -f k8s/service.yaml
                        if errorlevel 1 exit /b 1
                        kubectl set image deployment/hotel-booking hotel-booking="%APP_IMAGE%"
                        if errorlevel 1 exit /b 1
                        kubectl rollout status deployment/hotel-booking --timeout=180s
                    '''
                }
            }
        }
    }

    post {
        always {
            bat 'docker image rm "%APP_IMAGE%" >NUL 2>&1 || exit /b 0'
        }
    }
}
