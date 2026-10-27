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
                    sh 'echo $dockerhubpwd | docker login -u ssalmarezgui --password-stdin'
                }
                
                sh 'docker push ssalmarezgui/my-country-service:$BUILD_NUMBER'
            }
        }

        // Étape de déploiement Kubernetes ajoutée
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    kubeconfig(credentialsId: 'kubeconfig-file', serverUrl: '') {
                        sh 'kubectl apply -f deployment.yaml'
                        sh 'kubectl apply -f service.yaml'
                    }
                }
            }
        }
    }
}
