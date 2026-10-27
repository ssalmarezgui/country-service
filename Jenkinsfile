pipeline {
    agent any
    tools {
        maven 'M2_HOME'
        jdk 'JDK21'
    }
    stages {
        stage('Checkout code') {
            steps {
                checkout scm
            }
        }
        stage('Build maven') {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('Build Dockerfile') {
            steps {
                sh 'docker build -t majdaaa/my-compte-service:$BUILD_NUMBER .'
                withCredentials([string(credentialsId: 'dockerhub-paswd', variable: 'dockerhubpwd')]) {
                    sh 'echo $dockerhubpwd | docker login -u majdaaa --password-stdin'
                }
                retry(3) {
                    sh 'docker push majdaaa/my-compte-service:$BUILD_NUMBER'
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig(credentialsId: 'kubeconfig-file', serverUrl: '') {
                        sh 'kubectl apply -f deployment.yaml'
                        sh 'kubectl apply -f service.yaml'
                        sh 'kubectl set image deployment/my-country-service-deployement my-country-service=majdaaa/my-compte-service:$BUILD_NUMBER'
                        sh 'kubectl rollout status deployment/my-country-service-deployement'
                    }
                }
            }
        }
    }
}
