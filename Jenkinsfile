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
                    withCredentials([usernamePassword(
                        credentialsId: "dockerhub-${params.NAMESPACE}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh "docker build -t my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER} ."
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                        sh "docker tag my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER} ${DOCKER_USER}/my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER}"
                        retry(3) {
                            sh "docker push ${DOCKER_USER}/my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER}"
                        }
                    }
                }
            }
        }

        // --- ETAPE ANSIBLE VAULT (PAGE 2 DU COURS) ---
        stage('Deploy with Ansible') {
            steps {
                withCredentials([string(credentialsId: 'ansible_vault_pass', variable: 'VAULT_PASS')]) {
                    sh '''
                        echo "$VAULT_PASS" > vault_pass.txt
                        ansible-playbook -i localhost, -c local playbookCICD.yml --vault-password-file vault_pass.txt --check || true
                        rm -f vault_pass.txt
                    '''
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
                            withCredentials([usernamePassword(
                                credentialsId: "dockerhub-${params.NAMESPACE}",
                                usernameVariable: 'DOCKER_USER',
                                passwordVariable: 'DOCKER_PASS'
                            )]) {
                                sh "sed -i 's|IMAGE_PLACEHOLDER|${DOCKER_USER}/my-compte-service:${params.NAMESPACE}-${BUILD_NUMBER}|' deployment.yaml"
                            }

                            sh "kubectl create namespace ${params.NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"
                            sh "kubectl apply -f deployment.yaml -n ${params.NAMESPACE}"
                            sh "kubectl apply -f service.yaml -n ${params.NAMESPACE}"
                            sh "kubectl rollout status deployment/my-country-service-deployement -n ${params.NAMESPACE} --timeout=120s"
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Ansible playbook et pipeline executes avec succes !'
        }
    }
}
