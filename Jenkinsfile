pipeline {
    agent any

    environment {
        HELM_REPO_URL    = 'https://github.com/BenouadahAlaEddine/pratas-cd.git'
        K8S_NAMESPACE    = 'pratas'
        HELM_RELEASE     = 'pratas-app'
        KUBECONFIG_CREDS = 'k8s-kubeconfig'
    }

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Tag de l\'image à déployer')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Environnement cible')
    }

    stages {
        stage('📥 Checkout Helm Repo') {
            steps {
                git branch: 'main',
                    url: "${env.HELM_REPO_URL}",
                    credentialsId: 'gitlab-credentials'
            }
        }

        stage('⚙️ Setup Kubeconfig') {
            steps {
                withCredentials([file(credentialsId: env.KUBECONFIG_CREDS, variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        mkdir -p /var/lib/jenkins/.kube
                        cp $KUBECONFIG_FILE /var/lib/jenkins/.kube/config
                        chmod 600 /var/lib/jenkins/.kube/config
                    '''
                }
            }
        }

        stage('🚀 Helm Deploy') {
            steps {
                script {
                    def valuesFile = params.ENVIRONMENT == 'production' ? 'values-prod.yaml' : 'values.yaml'
                    def services = ['gateway', 'auth', 'products', 'orders', 'payments', 'notifications', 'frontend']
                    def setArgs = services.collect { svc ->
                        "--set ${svc}.image.tag=${params.IMAGE_TAG}"
                    }.join(" \\\n                            ")

                    sh """
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        helm upgrade --install ${env.HELM_RELEASE} . \\
                            --kubeconfig /var/lib/jenkins/.kube/config \\
                            --insecure-skip-tls-verify \\
                            --namespace ${env.K8S_NAMESPACE} \\
                            --create-namespace \\
                            -f ${valuesFile} \\
                            --set global.imageRegistry=docker.io/aladin78 \\
                            ${setArgs} \\
                            --wait \\
                            --timeout 5m \\
                            --atomic
                    """
                }
            }
        }

        stage('✅ Verify') {
            steps {
                sh """
                    export KUBECONFIG=/var/lib/jenkins/.kube/config

                    # Lister les vrais noms de deployments
                    echo "=== Deployments dans ${env.K8S_NAMESPACE} ==="
                    kubectl get deployments -n ${env.K8S_NAMESPACE}

                    # Rollout status sur tous les deployments du release
                    for dep in \$(kubectl get deployments -n ${env.K8S_NAMESPACE} \
                        -l app=${env.HELM_RELEASE} \
                        -o jsonpath='{.items[*].metadata.name}'); do
                        echo "Checking rollout: \$dep"
                        kubectl rollout status deployment/\$dep \
                            -n ${env.K8S_NAMESPACE} --timeout=3m
                    done

                    # Status final des pods
                    echo "=== Pods ==="
                    kubectl get pods -n ${env.K8S_NAMESPACE}
                """
            }
        }
    }

    post {
        success {
            echo "✅ Déploiement réussi sur ${params.ENVIRONMENT} avec le tag ${params.IMAGE_TAG}"
        }
        failure {
            sh """
                export KUBECONFIG=/var/lib/jenkins/.kube/config
                echo "=== Pods en échec ==="
                kubectl get pods -n ${env.K8S_NAMESPACE}
                echo "=== Events ==="
                kubectl get events -n ${env.K8S_NAMESPACE} \
                    --sort-by='.lastTimestamp' | tail -20
            """
            echo "❌ Échec du déploiement sur ${params.ENVIRONMENT}"
        }
    }
}