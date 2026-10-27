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
                    // Télécharge kubectl à la volée s'il n'est pas présent
                    sh '''
                        if ! command -v kubectl &> /dev/null; then
                            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                            chmod +x kubectl
                            mkdir -p bin
                            mv kubectl bin/
                        fi
                    '''
                    // Ajoute le dossier 'bin' au PATH pour trouver kubectl
                    withEnv(["PATH+KUBECTL=${WORKSPACE}/bin"]) {
                        kubeconfig(credentialsId: 'kubeconfig-file', serverUrl: '') {
                            // Remplacement du tag IMAGE_TAG par le numéro de build Jenkins
                            sh "sed -i 's/IMAGE_TAG/${BUILD_NUMBER}/g' deployment.yaml"
                            sh 'kubectl apply -f deployment.yaml'
                            sh 'kubectl apply -f service.yaml'
                            sh 'kubectl rollout status deployment/my-country-service-deployement'
                        }
                    }
                }
            }
        }
    }
}
