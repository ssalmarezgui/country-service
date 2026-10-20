pipeline {
    agent any
    tools {
        maven 'mymaven'
    }
    stages {
        stage('Checkout code') {
            steps {
                git branch: 'main', url: 'https://github.com/ssalmarezgui/country-service.git'
            }
        }
        stage('Build maven') {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('Build Dockerfile') {
            steps {
                sh 'docker build -t ssalmarezgui/my-country-service:$BUILD_NUMBER .'
                
                withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                    sh 'docker login -u ssalmarezgui -p ${dockerhubpwd}'
                }
                
                sh 'docker push ssalmarezgui/my-country-service:$BUILD_NUMBER'
            }
        }
        stage('Deploy micro-service') {
            steps {
                sh 'docker rm -f $(docker ps -aq) || true'
                sh 'docker run -d -p 8082:8082 ssalmarezgui/my-country-service:$BUILD_NUMBER'
            }
        }
    }
}
