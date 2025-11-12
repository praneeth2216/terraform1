pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-2'
        TF_VERSION = '1.6.0'
        AWS_CREDS = credentials('aws-creds')  // Jenkins credential ID
        TF_PLUGIN_CACHE_DIR = "$WORKSPACE/.terraform.d/plugin-cache"
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
                set -e
                if ! command -v terraform >/dev/null 2>&1; then
                    echo "Installing Terraform ${TF_VERSION}..."
                    wget -q https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip
                    unzip -o terraform_${TF_VERSION}_linux_amd64.zip
                    mkdir -p $HOME/bin
                    mv terraform $HOME/bin/
                    export PATH=$PATH:$HOME/bin
                    echo "Terraform installed locally at $HOME/bin"
                else
                    echo "Terraform already installed"
                fi
                terraform -version
                '''
            }
        }

        stage('Terraform Init') {
            steps {
                withEnv([
                    "AWS_ACCESS_KEY_ID=${AWS_CREDS_USR}",
                    "AWS_SECRET_ACCESS_KEY=${AWS_CREDS_PSW}"
                ]) {
                    sh '''
                    export PATH=$PATH:$HOME/bin
                    terraform init -backend-config="bucket=jenkinsmaster1" \
                                   -backend-config="key=dev/terraform.tfstate" \
                                   -backend-config="region=us-east-1" \
                                   -backend-config="dynamodb_table=Jenkins_master" \
                                   -backend-config="encrypt=true"
                    '''
                }
            }
        }

        stage('Terraform Validate') {
            steps {
                sh '''
                export PATH=$PATH:$HOME/bin
                terraform validate
                '''
            }
        }

        stage('Terraform Plan') {
            steps {
                withEnv([
                    "AWS_ACCESS_KEY_ID=${AWS_CREDS_USR}",
                    "AWS_SECRET_ACCESS_KEY=${AWS_CREDS_PSW}"
                ]) {
                    sh '''
                    export PATH=$PATH:$HOME/bin
                    terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Manual Approval') {
            steps {
                script {
                    def userInput = input(
                        id: 'approval',
                        message: 'Do you want to apply the Terraform changes?',
                        parameters: [choice(name: 'confirm', choices: 'yes\nno', description: 'Confirm apply')]
                    )
                    if (userInput == 'no') {
                        error("User aborted the deployment.")
                    }
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                withEnv([
                    "AWS_ACCESS_KEY_ID=${AWS_CREDS_USR}",
                    "AWS_SECRET_ACCESS_KEY=${AWS_CREDS_PSW}"
                ]) {
                    sh '''
                    export PATH=$PATH:$HOME/bin
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up workspace..."
            sh '''
            rm -rf terraform_${TF_VERSION}_linux_amd64.zip
            rm -rf .terraform
            '''
        }
        success {
            echo "✅ Terraform deployment completed successfully!"
        }
        failure {
            echo "❌ Terraform deployment failed!"
        }
    }
}
