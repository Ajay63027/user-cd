pipeline {
    agent { label 'AGENT-1' }
    environment { 
        REGION = "us-east-1"
        ACC_ID = "169189304039"
        PROJECT = "roboshop"
        COMPONENT = "user"
    }
    options {
        timeout(time: 30, unit: 'MINUTES') 
        disableConcurrentBuilds()
    }
    parameters {
        string(name: 'appVersion', description: 'Image version of the application')
        choice(name: 'deploy_to', choices: ['dev', 'qa', 'prod'], description: 'Pick the Environment')
    }

    stages {
        stage('Deploy') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {
                        sh """
                            aws eks update-kubeconfig --region $REGION --name "$PROJECT-${params.deploy_to}"
                            kubectl get nodes
                            kubectl apply -f 01-namespace.yaml
                            sed -i "s/IMAGE_VERSION/${params.appVersion}/g" values-${params.deploy_to}.yaml
                            helm upgrade --install $COMPONENT -f values-${params.deploy_to}.yaml -n $PROJECT .
                        """
                    }
                }
            }
        }

        stage('Check Rollout') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {
                        def deploymentStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/$COMPONENT --timeout=60s -n $PROJECT || echo FAILED").trim()
                        if (deploymentStatus.contains("successfully rolled out")) {
                            echo "Deployment Success ✅"
                        } else {
                            sh "helm rollback $COMPONENT -n $PROJECT"
                            error "Deployment failed ❌, rollback triggered"
                        }
                    }
                }
            }
        }

        stage('Functional Testing') {
            when { expression { params.deploy_to == "dev" } }
            steps { echo "Run functional test cases" }
        }

        stage('Integration Testing') {
            when { expression { params.deploy_to == "qa" } }
            steps { echo "Run integration test cases" }
        }

        stage('PROD Deploy Approval') {
            when { expression { params.deploy_to == "prod" } }
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {
                        sh """
                            echo "check CR, approvals, deployment window"
                            echo "then trigger PROD deploy"
                        """
                    }
                }
            }
        }
    }

    post { 
        always { 
            echo 'Cleaning up workspace...'
            deleteDir()
        }
        success { echo 'Pipeline Success 🎉' }
        failure { echo 'Pipeline Failed ❌' }
    }
}
