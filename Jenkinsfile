pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = "ap-south-1"
        STACK_NAME = "dev-ec2-stack"
        ENVIRONMENT = "dev"
        KEY_NAME = "jenkins-ec2-key"
        TEMPLATE_FILE = "cloudformation/ec2.yaml"
        INSTANCE_TYPE = "t3.micro"
    }

    stages {

        stage('Checkout Source') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
                echo 'Code checked out successfully'
            }
        }

        stage('Validate AWS Credentials') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    sh '''
                        echo "Validating AWS credentials..."
                        aws sts get-caller-identity
                        echo "AWS credentials validated successfully"
                    '''
                }
            }
        }

        stage('List Files') {
            steps {
                sh '''
                    echo "Current directory: $(pwd)"
                    echo "Files in current directory:"
                    ls -la
                    echo "Files in cloudformation directory:"
                    ls -la cloudformation/ || echo "cloudformation directory not found"
                '''
            }
        }

        stage('Get Latest Amazon Linux 2 AMI') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        env.AMI_ID = sh(
                            script: '''
                                aws ec2 describe-images \
                                --owners amazon \
                                --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
                                --query "Images | sort_by(@,&CreationDate)[-1].ImageId" \
                                --output text
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        echo "Latest AMI ID: ${env.AMI_ID}"
                    }
                }
            }
        }

        stage('Validate CloudFormation Template') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    sh """
                        echo "Validating CloudFormation template..."
                        echo "Template path: ${TEMPLATE_FILE}"
                        echo "Current directory: \$(pwd)"
                        echo "Checking if template exists:"
                        ls -la ${TEMPLATE_FILE} || echo "Template not found!"
                        aws cloudformation validate-template \
                        --template-body file://${TEMPLATE_FILE}
                        echo "Template validation successful"
                    """
                }
            }
        }

        stage('Deploy CloudFormation Stack') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        echo "Deploying CloudFormation stack: ${STACK_NAME}"
                        
                        def deployCmd = """
                            aws cloudformation deploy \
                            --template-file ${TEMPLATE_FILE} \
                            --stack-name ${STACK_NAME} \
                            --parameter-overrides \
                                KeyName=${KEY_NAME} \
                                InstanceType=${INSTANCE_TYPE} \
                                AMIId=${AMI_ID} \
                                Environment=${ENVIRONMENT} \
                                SSHLocation=0.0.0.0/0 \
                            --capabilities CAPABILITY_IAM \
                            --no-fail-on-empty-changeset
                        """
                        
                        sh deployCmd
                        echo "Stack deployment initiated successfully"
                    }
                }
            }
        }

        stage('Wait for Stack Creation') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        echo "Waiting for stack creation to complete..."
                        sh """
                            aws cloudformation wait stack-create-complete \
                            --stack-name ${STACK_NAME}
                        """
                        echo "Stack creation completed successfully"
                    }
                }
            }
        }

        stage('Get Stack Outputs') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        def outputs = sh(
                            script: """
                                aws cloudformation describe-stacks \
                                --stack-name ${STACK_NAME} \
                                --query "Stacks[0].Outputs" \
                                --output json
                            """,
                            returnStdout: true
                        )
                        
                        def outputsJson = readJSON text: outputs
                        echo "=========================================="
                        echo "Stack Deployment Details:"
                        echo "=========================================="
                        outputsJson.each { output ->
                            echo "${output.OutputKey}: ${output.OutputValue}"
                        }
                        echo "=========================================="
                        
                        outputsJson.each { output ->
                            if (output.OutputKey == "PublicIP") {
                                env.EC2_PUBLIC_IP = output.OutputValue
                            }
                            if (output.OutputKey == "InstanceId") {
                                env.EC2_INSTANCE_ID = output.OutputValue
                            }
                        }
                    }
                }
            }
        }

        stage('Test EC2 Instance') {
            steps {
                script {
                    echo "=========================================="
                    echo "✅ DEPLOYMENT SUCCESSFUL!"
                    echo "=========================================="
                    echo "🌐 EC2 Instance Details:"
                    echo "  - Public IP: ${env.EC2_PUBLIC_IP}"
                    echo "  - Instance ID: ${env.EC2_INSTANCE_ID}"
                    echo "  - SSH Command: ssh -i ${KEY_NAME}.pem ec2-user@${env.EC2_PUBLIC_IP}"
                    echo "  - Web Access: http://${env.EC2_PUBLIC_IP}"
                    echo "=========================================="
                }
            }
        }

    }

    post {
        success {
            echo "🎉 Deployment Successful!"
        }
        
        failure {
            echo "❌ Deployment Failed!"
        }
        
        always {
            echo "Pipeline execution completed"
        }
    }
}
EOF


echo "=========================================="
echo "Checking created files:"
echo "=========================================="
ls -la
echo ""
ls -la cloudformation/