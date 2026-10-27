pipeline {
    agent any

    parameters {
        choice(
            name: 'NAMESPACE',
            choices: ['salma-env', 'majda-env'],
            description: 'Qui lance le déploiement ?'
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
                script {
                    // Récupère automatiquement les identifiants de la bonne personne (dockerhub-salma-env ou dockerhub-majda-env)
                    withCredentials([usernamePassword(
                        credentialsId: "dockerhub-${params.NAMESPACE}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        // 1. Build de l'image
                        sh "docker build -t my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER} ."

                        // 2. Connexion avec les identifiants de celle qui a lancé le build
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'

                        // 3. Tag vers le compte Docker de celle qui a lancé le build
                        sh "docker tag my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER} ${DOCKER_USER}/my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER}"

                        // 4. Push
                        retry(3) {
                            sh "docker push ${DOCKER_USER}/my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER}"
                        }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh '''
                        if ! command -v kubectl &> /dev/null; then
                            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                            chmod +x kubectl
                            mkdir -p bin
                            mv kubectl bin/
                        fi
                    '''

                    withEnv(["PATH+KUBECTL=${WORKSPACE}/bin"]) {
                        kubeconfig(credentialsId: 'kubeconfig-file', serverUrl: '') {
                            
                            // On réutilise le DOCKER_USER pour injecter la bonne image dans le yaml
                            withCredentials([usernamePassword(
                                credentialsId: "dockerhub-${params.NAMESPACE}",
                                usernameVariable: 'DOCKER_USER',
                                passwordVariable: 'DOCKER_PASS'
                            )]) {
                                sh "sed -i 's|IMAGE_PLACEHOLDER|${DOCKER_USER}/my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER}|' deployment.yaml"
                            }

                            // Crée le namespace s'il n'existe pas
                            sh "kubectl create namespace ${params.NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"

                            // Déploie dans le bon namespace
                            sh "kubectl apply -f deployment.yaml -n ${params.NAMESPACE}"
                            sh "kubectl apply -f service.yaml -n ${params.NAMESPACE}"

                            sh "kubectl rollout status deployment/my-country-service-deployement -n ${params.NAMESPACE} --timeout=120s"
                        }
                    }
                }
            }
        }
    }
}
