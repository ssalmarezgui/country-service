pipeline {
    agent any

    // 1. Permet de choisir l'environnement lors du build (Build with Parameters)
    parameters {
        choice(
            name: 'NAMESPACE',
            choices: ['majda-env', 'salma-env'],
            description: 'Sélectionnez l\'environnement cible pour le déploiement'
        )
    }

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
                // On inclut le namespace dans le tag Docker pour éviter d'écraser l'image de l'autre
                sh "docker build -t my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER} ."
                
                withCredentials([string(credentialsId: 'dockerhub-paswd', variable: 'dockerhubpwd')]) {
                    sh 'echo $dockerhubpwd | docker login -u majdaaa --password-stdin'
                }
                
                sh "docker tag my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER} majdaaa/my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER}"
                
                retry(3) {
                    sh "docker push majdaaa/my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER}"
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
                            
                            // 1. Remplacement de l'image avec le bon tag
                            sh "sed -i 's|IMAGE_PLACEHOLDER|majdaaa/my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER}|' deployment.yaml"

                            // 2. Crée le namespace automatiquement s'il n'existe pas encore
                            sh "kubectl create namespace ${params.NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"

                            // 3. Application des fichiers dans le namespace sélectionné
                            sh "kubectl apply -f deployment.yaml -n ${params.NAMESPACE}"
                            sh "kubectl apply -f service.yaml -n ${params.NAMESPACE}"

                            // 4. Suivi de l'état du déploiement
                            sh "kubectl rollout status deployment/my-country-service-deployement -n ${params.NAMESPACE} --timeout=120s"
                        }
                    }
                }
            }
        }
    }
}
