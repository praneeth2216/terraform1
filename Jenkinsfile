pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-2'
        TF_VERSION = '1.6.0'
        TF_HOME = '/usr/local/bin'
        // Jenkins credentials ID containing AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
        AWS_CREDS = credentials('aws-creds')
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out Terraform code..."
                checkout scm
            }
        }

        stage('Install Terraform') {
            steps {
                sh '''
                if ! terraform -version | grep -q ${TF_VERSION}; then
                    echo "Installing Terraform ${TF_VERSION}..."
                    wget -q https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip
                    unzip -o terraform_${TF_VERSION}_linux_amd64.zip
                    sudo mv terraform ${TF_HOME}/terraform
                else
                    echo "Terraform already installed"
                fi
                '''
            }
        }

        stage('Terraform Init') {
            steps {
                withEnv(["AWS_ACCESS_KEY_ID=${AWS_CREDS_USR}", "AWS_SECRET_ACCESS_KEY=${AWS_CREDS_PSW}"]) {
                    sh '''
                    echo "Initializing Terraform backend..."
                    terraform init -backend-config="bucket=jenkinsmaster1" \
                                   -backend-config="key=dev/terraform.tfstate" \
                                   -backend-config="region=us-east-1" \
                                   -backend-config="dynamodb_table=Jenkins_master"
                    '''
                }
            }
        }

        stage('Terraform Validate') {
            steps {
                sh 'terraform validate'
            }
        }

        stage('Terraform Plan') {
            steps {
                withEnv(["AWS_ACCESS_KEY_ID=${AWS_CREDS_USR}", "AWS_SECRET_ACCESS_KEY=${AWS_CREDS_PSW}"]) {
                    sh 'terraform plan -out=tfplan'
                }
            }
        }

        stage('Manual Approval') {
            steps {
                script {
                    def userInput = input(
                        id: 'Proceed1', message: 'Apply Terraform changes?', parameters: [
                            choice(choices: 'yes\nno', description: 'Deploy changes?', name: 'approve')
                        ]
                    )
                    if (userInput == 'no') {
                        error("User cancelled deployment.")
                    }
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                withEnv(["AWS_ACCESS_KEY_ID=${AWS_CREDS_USR}", "AWS_SECRET_ACCESS_KEY=${AWS_CREDS_PSW}"]) {
                    sh 'terraform apply -auto-approve tfplan'
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline complete."
        }
        success {
            echo "✅ Terraform deployment successful!"
        }
        failure {
            echo "❌ Terraform deployment failed."
        }
    }
}
