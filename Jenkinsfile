pipeline {
    agent any

    tools {
        maven 'mymaven'
    }

    environment {
        // A adapter
        SONAR_SERVER   = 'SonarQube'          // Manage Jenkins > System > SonarQube servers (nom)
        VAULT_CRED_ID  = 'ansible-vault-pass' // Credential Jenkins (Secret text) = mot de passe de vault.yml
        APP_LABEL      = 'countryproject'     // label "app:" de votre deployment.yaml
        K8S_NAMESPACE  = 'default'
        // Credential "Secret file" existant : Jenkins cree un fichier temporaire et
        // exporte son chemin dans KUBECONFIG (utilise par kubectl, helm et ansible)
        KUBECONFIG     = credentials('kubeconfig-file')
    }

    stages {

        stage('Checkout code') {
            steps {
                git branch: 'master', url: 'https://github.com/amenibejaoui/countryproject.git'
            }
        }

        stage('Compile code') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test code') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv("${SONAR_SERVER}") {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        // Optionnel (necessite le webhook SonarQube -> Jenkins)
        // stage('Quality Gate') {
        //     steps {
        //         timeout(time: 5, unit: 'MINUTES') {
        //             waitForQualityGate abortPipeline: true
        //         }
        //     }
        // }

        stage('Package code') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        // Ansible : build de l'image, push sur Docker Hub, deploiement sur Kubernetes
        stage('Deploy with Ansible') {
            steps {
                withCredentials([string(credentialsId: "${VAULT_CRED_ID}", variable: 'VAULT_PASS')]) {
                    sh '''
                        trap 'rm -f .vault_pass' EXIT
                        echo "$VAULT_PASS" > .vault_pass
                        ansible-playbook playbookCICD.yml --vault-password-file .vault_pass
                    '''
                }
            }
        }

        stage('Verify pods') {
            steps {
                sh '''
                    kubectl rollout status deployment -l app=${APP_LABEL} -n ${K8S_NAMESPACE} --timeout=180s
                    kubectl get pods -l app=${APP_LABEL} -n ${K8S_NAMESPACE} -o wide
                '''
            }
        }

        // Monitoring : kube-state-metrics (idempotent grace a "upgrade --install")
        stage('Monitoring setup') {
            steps {
                sh '''
                    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                    helm repo update
                    helm upgrade --install kube-state-metrics prometheus-community/kube-state-metrics \
                        --namespace monitoring --create-namespace \
                        --set service.type=NodePort \
                        --set service.nodePort=30080
                    kubectl get pods -n monitoring
                '''
            }
        }
    }

    post {
        success { echo 'Pipeline termine : application deployee et metriques exposees.' }
        failure { echo 'Pipeline en echec : consulter les logs de l etape en erreur.' }
    }
}
