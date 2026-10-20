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
                sh 'docker build -t ssalmarezgui/my-country-service:$BUILD_NUMBER .'
                
                withCredentials([string(credentialsId: 'dockerhub-paswd', variable: 'dockerhubpwd')]) {
                    sh "echo '\$dockerhubpwd' | docker login -u ssalmarezgui --password-stdin"
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
