pipeline {
    agent any
    
    parameters {
        string(name: 'STACK_NAME', defaultValue: 'prometheus-exporters-stack', description: 'Name of the CloudFormation stack')
        string(name: 'TEMPLATE_PATH', defaultValue: 'jenkins_machin.yaml', description: 'Path to the CloudFormation template file')
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS Region to deploy the stack')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Validate Template') {
            steps {
                script {
                    sh """
                    aws cloudformation validate-template \
                        --template-body file://./${params.TEMPLATE_PATH} \
                        --region ${params.AWS_REGION}
                    """
                }
            }
        }
        
        stage('Deploy Stack') {
            steps {
                script {
                    def stackExists = sh(
                        script: "aws cloudformation describe-stacks --stack-name ${params.STACK_NAME} --region ${params.AWS_REGION} 2>&1 || echo 'STACK_NOT_FOUND'",
                        returnStdout: true
                    ).contains('STACK_NOT_FOUND') ? false : true
                    
                    if (stackExists) {
                        echo "Stack ${params.STACK_NAME} exists. Updating..."
                        sh """
                            aws cloudformation update-stack \
                                --stack-name ${params.STACK_NAME} \
                                --template-body file://${params.TEMPLATE_PATH} \
                                --region ${params.AWS_REGION} \
                                --capabilities CAPABILITY_IAM || echo 'No updates are to be performed'
                        """
                    } else {
                        echo "Stack ${params.STACK_NAME} does not exist. Creating..."
                        sh """
                            aws cloudformation create-stack \
                                --stack-name ${params.STACK_NAME} \
                                --template-body file://${params.TEMPLATE_PATH} \
                                --region ${params.AWS_REGION} \
                                --capabilities CAPABILITY_IAM
                        """
                    }
                }
            }
        }
        
        stage('Wait for Stack Completion') {
            steps {
                script {
                    def stackExists = sh(
                        script: "aws cloudformation describe-stacks --stack-name ${params.STACK_NAME} --region ${params.AWS_REGION} 2>&1 || echo 'STACK_NOT_FOUND'",
                        returnStdout: true
                    ).contains('STACK_NOT_FOUND') ? false : true
                    
                    if (stackExists) {
                        echo "Waiting for stack operation to complete..."
                        sh """
                            aws cloudformation wait stack-update-complete --stack-name ${params.STACK_NAME} --region ${params.AWS_REGION} || \
                            aws cloudformation wait stack-create-complete --stack-name ${params.STACK_NAME} --region ${params.AWS_REGION}
                        """
                    }
                }
            }
        }
        
        stage('Get Stack Outputs') {
            steps {
                script {
                    echo "Getting stack outputs..."
                    sh """
                        aws cloudformation describe-stacks \
                            --stack-name ${params.STACK_NAME} \
                            --query 'Stacks[0].Outputs' \
                            --region ${params.AWS_REGION} \
                            --output table
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo "Pipeline executed successfully! The CloudFormation stack has been created or updated."
        }
        failure {
            echo "Pipeline execution failed. Check the logs for details."
        }
    }
}
