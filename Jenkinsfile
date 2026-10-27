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
                sh 'docker build -t my-compte-service:$BUILD_NUMBER .'
                withCredentials([string(credentialsId: 'dockerhub-paswd', variable: 'dockerhubpwd')]) {
                    sh 'echo $dockerhubpwd | docker login -u majdaaa --password-stdin'
                }
                sh 'docker tag my-compte-service:$BUILD_NUMBER majdaaa/my-compte-service:$BUILD_NUMBER'
                retry(3) {
                    sh 'docker push majdaaa/my-compte-service:$BUILD_NUMBER'
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    kubeconfig(credentialsId: 'kubeconfig-file', serverUrl: '') {
                        sh "sed -i 's/IMAGE_TAG/${BUILD_NUMBER}/' deployment.yaml"
                        sh 'kubectl apply -f deployment.yaml'
                        sh 'kubectl apply -f service.yaml'
                        sh 'kubectl rollout status deployment/my-country-service-deployement'
                    }
                }
            }
        }
    }
}
