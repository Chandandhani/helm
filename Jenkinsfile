pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['DEPLOY', 'ROLLBACK', 'TEMPLATE_ONLY', 'LINT_ONLY'],
            description: 'Choose Deployment or Validation Action'
        )

        string(
            name: 'K8S_TARGET_NAMESPACE',
            defaultValue: 'default',
            description: 'Target Kubernetes Namespace for Execution'
        )

        string(
            name: 'ROLLBACK_REVISION',
            defaultValue: '',
            description: 'Helm Revision Number For Rollback'
        )

        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'uat', 'prod'],
            description: 'Deployment Environment (Used to pull matching values file)'
        )

        string(
            name: 'IMAGE_TAG',
            defaultValue: '',
            description: 'Docker Image Tag'
        )
    }

    environment {
        APP_NAME         = "release-pilot"
        RELEASE_NAME     = "payment-${params.ENVIRONMENT}"
        K8S_NAMESPACE    = "${params.K8S_TARGET_NAMESPACE}"
        
        HELM_REPO_NAME   = "release-pilot"
        HELM_REPO_URL    = "https://chandandhani.github.io/helm"
        HELM_CHART       = "release-pilot/release-pilot"
        IMAGE_REPOSITORY = "adamtravis/rollouts"
        
        // REMOVED hardcoded absolute file string variable block path
    }

    stages {
        stage('Workspace Cleanup') {
            steps {
                cleanWs()
            }
        }

        stage('Download Values File') {
            when {
                expression {
                    return params.ACTION in ['DEPLOY', 'TEMPLATE_ONLY', 'LINT_ONLY']
                }
            }
            steps {
                sh """
                echo "Downloading ${params.ENVIRONMENT} values file"
                curl -L -o ${params.ENVIRONMENT}-values.yaml \
                https://raw.githubusercontent.com/Chandandhani/helm/main/release-pilot/env/${params.ENVIRONMENT}-values.yaml
                ls -l
                """
            }
        }

        stage('Add Helm Repo') {
            steps {
                sh """
                helm repo add ${HELM_REPO_NAME} ${HELM_REPO_URL} || true
                helm repo update
                """
            }
        }

        stage('Helm Lint') {
            when {
                expression {
                    return params.ACTION in ['DEPLOY', 'LINT_ONLY', 'TEMPLATE_ONLY']
                }
            }
            steps {
                sh """
                helm pull ${HELM_CHART} --untar
                helm lint release-pilot -f ${params.ENVIRONMENT}-values.yaml
                """
            }
        }

        stage('Helm Template (Dry-Run)') {
            when {
                expression {
                    return params.ACTION in ['DEPLOY', 'TEMPLATE_ONLY']
                }
            }
            steps {
                sh """
                echo "Generating Manifest Dry-Run Output via Helm Template..."
                helm template ${RELEASE_NAME} release-pilot \
                  --namespace ${K8S_NAMESPACE} \
                  -f ${params.ENVIRONMENT}-values.yaml \
                  --set image.repository=${IMAGE_REPOSITORY} \
                  --set image.tag=${params.IMAGE_TAG}
                """
            }
        }

        stage('Helm Deploy / Upgrade') {
            when {
                expression {
                    params.ACTION == 'DEPLOY'
                }
            }
            steps {
                // Securely extract the file credential store and map it straight to KUBECONFIG
                configFileProvider([
                    configFile(fileId: '4427e3f3-6109-4b15-92cd-fb2900c7e0b7', variable: 'KUBECONFIG')
                ]) {
                    sh """
                    echo "Starting Helm Deployment"
                    helm upgrade --install ${RELEASE_NAME} ${HELM_CHART} \
                      --namespace ${K8S_NAMESPACE} \
                      --create-namespace \
                      -f ${params.ENVIRONMENT}-values.yaml \
                      --set image.repository=${IMAGE_REPOSITORY} \
                      --set image.tag=${params.IMAGE_TAG} \
                      --wait \
                      --atomic \
                      --timeout 5m
                    """
                }
            }
        }

        stage('Verify Rollout') {
            when {
                expression {
                    params.ACTION == 'DEPLOY'
                }
            }
            steps {
                configFileProvider([
                    configFile(fileId: '4427e3f3-6109-4b15-92cd-fb2900c7e0b7', variable: 'KUBECONFIG')
                ]) {
                    sh """
                    kubectl rollout status deployment/${RELEASE_NAME}-release-pilot \
                    -n ${K8S_NAMESPACE} \
                    --timeout=300s
                    """
                }
            }
        }

        stage('Helm History') {
            when {
                expression {
                    return params.ACTION in ['DEPLOY', 'ROLLBACK']
                }
            }
            steps {
                configFileProvider([
                    configFile(fileId: '4427e3f3-6109-4b15-92cd-fb2900c7e0b7', variable: 'KUBECONFIG')
                ]) {
                    sh """
                    helm history ${RELEASE_NAME} \
                    -n ${K8S_NAMESPACE} || true
                    """
                }
            }
        }

        stage('Rollback Deployment') {
            when {
                expression {
                    params.ACTION == 'ROLLBACK'
                }
            }
            steps {
                configFileProvider([
                    configFile(fileId: '4427e3f3-6109-4b15-92cd-fb2900c7e0b7', variable: 'KUBECONFIG')
                ]) {
                    sh """
                    echo "Starting Rollback"
                    helm rollback ${RELEASE_NAME} ${ROLLBACK_REVISION} \
                      -n ${K8S_NAMESPACE} \
                      --wait \
                      --timeout 5m
                    """
                }
            }
        }

        stage('Verify Rollback') {
            when {
                expression {
                    params.ACTION == 'ROLLBACK'
                }
            }
            steps {
                configFileProvider([
                    configFile(fileId: '4427e3f3-6109-4b15-92cd-fb2900c7e0b7', variable: 'KUBECONFIG')
                ]) {
                    sh """
                    kubectl rollout status deployment/${RELEASE_NAME}-release-pilot \
                    -n ${K8S_NAMESPACE} \
                    --timeout=300s
                    """
                }
            }
        }
    }

    post {
        success {
            echo "===================================="
            echo "Pipeline Execution Successful"
            echo "===================================="
            
            script {
                if (params.ACTION in ['DEPLOY', 'ROLLBACK']) {
                    configFileProvider([
                        configFile(fileId: '4427e3f3-6109-4b15-92cd-fb2900c7e0b7', variable: 'KUBECONFIG')
                    ]) {
                        sh "kubectl get pods -n ${K8S_NAMESPACE}"
                    }
                }
            }
        }

        failure {
            echo "===================================="
            echo "Pipeline Failed"
            echo "===================================="
            script {
                if (params.ACTION in ['DEPLOY', 'ROLLBACK']) {
                    configFileProvider([
                        configFile(fileId: '4427e3f3-6109-4b15-92cd-fb2900c7e0b7', variable: 'KUBECONFIG')
                    ]) {
                        sh "helm history ${RELEASE_NAME} -n ${K8S_NAMESPACE} || true"
                    }
                }
            }
        }

        always {
            echo "===================================="
            echo "Pipeline Completed"
            echo "===================================="
            cleanWs()
        }
    }
}
