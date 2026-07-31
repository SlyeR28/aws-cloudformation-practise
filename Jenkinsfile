pipeline {
    // ↑ The entire automation pipeline
    
    agent any
    // ↑ Run on any available Jenkins worker

    environment {
        // ↓ Environment variables used in all stages
        AWS_DEFAULT_REGION = "ap-south-1"
        // ↑ AWS Mumbai region (choose closest to you)
        
        STACK_NAME = "dev-ec2-stack"
        // ↑ Name of the CloudFormation stack
        
        ENVIRONMENT = "dev"
        // ↑ Environment (dev/staging/prod)
        
        KEY_NAME = "jenkins-ec2-key"
        // ↑ Name of the SSH key pair
        
        TEMPLATE_FILE = "cloudformationTempletes/ec2.yaml"
        // ↑ Path to the CloudFormation template
        
        INSTANCE_TYPE = "t3.micro"
        // ↑ Free Tier eligible instance type
    }

    stages {
        // ↓ Each stage is a step in the deployment process

        // === STAGE 1: Get Code from GitHub ===
        stage('Checkout Source') {
            steps {
                echo '📥 Checking out code from GitHub...'
                checkout scm
                // ↑ Downloads the code from GitHub
                echo '✅ Code checked out successfully'
            }
        }

        // === STAGE 2: Verify AWS Credentials ===
        stage('Validate AWS Credentials') {
            steps {
                withCredentials([
                    // ↑ Use AWS credentials from Jenkins
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                        // ↑ Must match the ID in Jenkins
                    ]
                ]) {
                    sh '''
                        echo "🔐 Validating AWS credentials..."
                        aws sts get-caller-identity
                        // ↑ Gets who you're logged in as
                        echo "✅ AWS credentials validated successfully"
                    '''
                }
            }
        }

        // === STAGE 3: List Files (Debugging) ===
        stage('List Files') {
            steps {
                sh '''
                    echo "📂 Current directory: $(pwd)"
                    echo "Files in current directory:"
                    ls -la
                    echo "Files in cloudformation directory:"
                    ls -la cloudformation/ || echo "cloudformation directory not found"
                '''
            }
        }

        // === STAGE 4: Get Latest AMI ===
        stage('Get Latest Amazon Linux 2 AMI') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        // ↓ Find the newest Amazon Linux image
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
                        // ↑ This gets the latest Linux AMI
                        
                        echo "🖼️ Latest AMI ID: ${env.AMI_ID}"
                    }
                }
            }
        }

        // === STAGE 5: Validate Template ===
        stage('Validate CloudFormation Template') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    sh """
                        echo "✅ Validating CloudFormation template..."
                        echo "Template path: ${TEMPLATE_FILE}"
                        echo "Current directory: \$(pwd)"
                        echo "Checking if template exists:"
                        ls -la ${TEMPLATE_FILE} || echo "Template not found!"
                        aws cloudformation validate-template \
                        --template-body file://${TEMPLATE_FILE}
                        // ↑ AWS checks if the template is valid
                        echo "✅ Template validation successful"
                    """
                }
            }
        }

        // === STAGE 6: Check if Stack Already Exists ===
        stage('Check if Stack Exists') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        def stackExists = sh(
                            script: """
                                aws cloudformation describe-stacks \
                                --stack-name ${STACK_NAME} \
                                --query "Stacks[0].StackStatus" \
                                --output text 2>/dev/null || echo "NOT_FOUND"
                            """,
                            returnStdout: true
                        ).trim()
                        
                        env.STACK_EXISTS = stackExists != "NOT_FOUND"
                        echo "📊 Stack exists: ${env.STACK_EXISTS}"
                        if (env.STACK_EXISTS) {
                            echo "Current stack status: ${stackExists}"
                        }
                    }
                }
            }
        }

        // === STAGE 7: Delete Old Stack (if exists) ===
        stage('Delete Existing Stack (if any)') {
            // ↓ Only run this stage if stack exists
            when {
                expression { env.STACK_EXISTS == "true" }
            }
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        echo "🗑️ Deleting existing stack: ${STACK_NAME}"
                        sh """
                            aws cloudformation delete-stack \
                            --stack-name ${STACK_NAME}
                        """
                        
                        echo "⏳ Waiting for stack deletion to complete..."
                        sh """
                            aws cloudformation wait stack-delete-complete \
                            --stack-name ${STACK_NAME}
                        """
                        echo "✅ Stack deleted successfully"
                    }
                }
            }
        }

        // === STAGE 8: Deploy Stack ===
        stage('Deploy CloudFormation Stack') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        echo "🚀 Deploying CloudFormation stack: ${STACK_NAME}"
                        
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
                        // ↑ Deploys the stack with parameters
                        
                        sh deployCmd
                        echo "✅ Stack deployment initiated successfully"
                    }
                }
            }
        }

        // === STAGE 9: Wait for Completion ===
        stage('Wait for Stack Creation') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        echo "⏳ Waiting for stack creation to complete..."
                        sh """
                            aws cloudformation wait stack-create-complete \
                            --stack-name ${STACK_NAME}
                        """
                        // ↑ Pauses until stack is ready
                        echo "✅ Stack creation completed successfully"
                    }
                }
            }
        }

        // === STAGE 10: Get Stack Outputs ===
        stage('Get Stack Outputs') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        // ↓ Get the outputs (IP address, etc.)
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
                        echo "📋 STACK DEPLOYMENT DETAILS"
                        echo "=========================================="
                        outputsJson.each { output ->
                            echo "${output.OutputKey}: ${output.OutputValue}"
                        }
                        echo "=========================================="
                        
                        // ↓ Save important outputs as variables
                        outputsJson.each { output ->
                            if (output.OutputKey == "PublicIP") {
                                env.EC2_PUBLIC_IP = output.OutputValue
                            }
                            if (output.OutputKey == "InstanceId") {
                                env.EC2_INSTANCE_ID = output.OutputValue
                            }
                            if (output.OutputKey == "PublicDNS") {
                                env.EC2_PUBLIC_DNS = output.OutputValue
                            }
                        }
                    }
                }
            }
        }

        // === STAGE 11: Summary ===
        stage('Deployment Summary') {
            steps {
                script {
                    echo "=========================================="
                    echo "✅ DEPLOYMENT SUCCESSFUL!"
                    echo "=========================================="
                    echo "🌐 EC2 Instance Details:"
                    echo "  - Public IP: ${env.EC2_PUBLIC_IP}"
                    echo "  - Public DNS: ${env.EC2_PUBLIC_DNS}"
                    echo "  - Instance ID: ${env.EC2_INSTANCE_ID}"
                    echo "  - SSH Command: ssh -i ${KEY_NAME}.pem ec2-user@${env.EC2_PUBLIC_IP}"
                    echo "  - Web Access: http://${env.EC2_PUBLIC_IP}"
                    echo "=========================================="
                    echo "🏷️ Environment: ${ENVIRONMENT}"
                    echo "📌 Region: ${AWS_DEFAULT_REGION}"
                    echo "=========================================="
                }
            }
        }

    }

    // === POST-PIPELINE ACTIONS ===
    post {
        // What to do when pipeline succeeds
        success {
            echo "=========================================="
            echo "🎉 DEPLOYMENT SUCCESSFUL!"
            echo "=========================================="
            echo "Stack: ${STACK_NAME}"
            echo "Environment: ${ENVIRONMENT}"
            echo "Region: ${AWS_DEFAULT_REGION}"
            echo "Instance IP: ${env.EC2_PUBLIC_IP}"
            echo "=========================================="
        }
        
        // What to do when pipeline fails
        failure {
            echo "=========================================="
            echo "❌ DEPLOYMENT FAILED!"
            echo "=========================================="
            echo "Stack: ${STACK_NAME}"
            
            withCredentials([
                [
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials'
                ]
            ]) {
                script {
                    // Get error details from CloudFormation
                    def events = sh(
                        script: """
                            aws cloudformation describe-stack-events \
                            --stack-name ${STACK_NAME} \
                            --max-items 5 \
                            --query "StackEvents[?ResourceStatus=='CREATE_FAILED' || ResourceStatus=='UPDATE_FAILED' || ResourceStatus=='ROLLBACK_COMPLETE']" \
                            --output table
                        """,
                        returnStdout: true
                    )
                    echo "📋 Recent stack events:\n${events}"
                }
            }
            
            echo "Check CloudFormation console for detailed error information"
            echo "=========================================="
        }
        
        // Always run this
        always {
            echo "Pipeline execution completed"
        }
    }
}
