pipeline {
    agent any

    environment {
        HELM_REPO_URL    = 'https://github.com/BenouadahAlaEddine/pratas-cd.git' // Ton repo Helm
        K8S_NAMESPACE    = 'pratas'
        HELM_RELEASE     = 'pratas-app'
        KUBECONFIG_CREDS = 'k8s-kubeconfig'
    }

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Tag de l\'image à déployer (ex: 15-a1b2c3d)')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Environnement cible')
    }

    stages {
        // ── Stage 1: Checkout Helm Charts ─────────────────────────────────────
        stage('📥 Checkout Helm Repo') {
            steps {
                git branch: 'main', 
                    url: "${env.HELM_REPO_URL}", 
                    credentialsId: 'gitlab-credentials' // Ou github-credentials
            }
        }

        // ── Stage 2: Configure Kubectl ────────────────────────────────────────
        stage('⚙️ Setup Kubeconfig') {
            steps {
                script {
                    withCredentials([file(credentialsId: env.KUBECONFIG_CREDS, variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            mkdir -p $HOME/.kube
                            cp $KUBECONFIG_FILE $HOME/.kube/config
                            chmod 600 $HOME/.kube/config
                        '''
                    }
                }
            }
        }

        // ── Stage 3: Deploy with Helm ─────────────────────────────────────────
        stage('🚀 Helm Deploy') {
            steps {
                script {
                    withCredentials([file(credentialsId: env.KUBECONFIG_CREDS, variable: 'KUBECONFIG_FILE')]) {
                        def valuesFile = params.ENVIRONMENT == 'production' ? 'values-prod.yaml' : 'values.yaml'
                        
                        def services = ['gateway', 'auth', 'products', 'orders', 'payments', 'notifications', 'frontend']
                        def setArgs = ""
                        services.each { svc ->
                            setArgs += " --set ${svc}.image.tag=${params.IMAGE_TAG}"
                        }

                        sh """
                            helm upgrade --install ${env.HELM_RELEASE} . \\
                                --kubeconfig \$KUBECONFIG_FILE \\
                                --namespace ${env.K8S_NAMESPACE} \\
                                --create-namespace \\
                                -f ${valuesFile} \\
                                --set global.imageRegistry="docker.io/aladin78" \\
                                ${setArgs} \\
                                --wait \\
                                --timeout 5m \\
                                --atomic
                        """
                    }
                }
            }
        }

        // ── Stage 4: Verify Deployment ────────────────────────────────────────
        stage('✅ Verify') {
            steps {
                sh "kubectl rollout status deployment/${env.HELM_RELEASE}-gateway -n ${env.K8S_NAMESPACE} --timeout=3m"
                sh "kubectl get pods -n ${env.K8S_NAMESPACE}"
            }
        }
    }

    post {
        success {
            echo "✅ Déploiement réussi sur ${params.ENVIRONMENT} avec le tag ${params.IMAGE_TAG}"
        }
        failure {
            echo "❌ Échec du déploiement."
        }
    }
}