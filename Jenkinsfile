pipeline {
    // ↑ The entire automation pipeline
    
    agent any
    // ↑ Run on any available Jenkins worker

    environment {
        // ↓ Environment variables used in all stages
        AWS_DEFAULT_REGION = "ap-south-1"
        // ↑ AWS Mumbai region
        
        STACK_NAME = "ubuntu-ec2-stack"
        // ↑ Name of the CloudFormation stack
        
        ENVIRONMENT = "dev"
        // ↑ Environment (dev/staging/prod)
        
        KEY_NAME = "jenkins-ec2-key.pem"
        // ↑ Name of the SSH key pair (without .pem extension)
        
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
                
                // List files for debugging
                sh '''
                    echo "📂 Repository contents:"
                    ls -la
                    echo ""
                    echo "📂 CloudFormation templates:"
                    find . -name "*.yaml" -o -name "*.yml" | head -10
                '''
            }
        }

        // === STAGE 2: Validate AWS Credentials ===
        stage('Validate AWS Credentials') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                        // ↑ Must match the ID in Jenkins
                    ]
                ]) {
                    sh '''
                        echo "🔐 Validating AWS credentials..."
                        aws sts get-caller-identity
                        echo "✅ AWS credentials validated successfully"
                        echo "📍 Region: ap-south-1 (Mumbai)"
                    '''
                }
            }
        }

        // === STAGE 3: Get Latest Ubuntu AMI ===
        stage('Get Latest Ubuntu 22.04 AMI') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        echo "🖼️ Fetching latest Ubuntu 22.04 AMI for Mumbai region..."
                        
                        env.UBUNTU_AMI_ID = sh(
                            script: '''
                                aws ec2 describe-images \
                                --region ap-south-1 \
                                --owners 099720109477 \
                                --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
                                --query "Images | sort_by(@,&CreationDate)[-1].ImageId" \
                                --output text
                            ''',
                            returnStdout: true
                        ).trim()
                        // ↑ This gets the latest Ubuntu 22.04 AMI in Mumbai
                        
                        echo "🖼️ Latest Ubuntu AMI ID: ${env.UBUNTU_AMI_ID}"
                        
                        // Also get Ubuntu 24.04 as backup option
                        env.UBUNTU_24_AMI_ID = sh(
                            script: '''
                                aws ec2 describe-images \
                                --region ap-south-1 \
                                --owners 099720109477 \
                                --filters "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
                                --query "Images | sort_by(@,&CreationDate)[-1].ImageId" \
                                --output text
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        echo "🖼️ Latest Ubuntu 24.04 AMI ID (backup): ${env.UBUNTU_24_AMI_ID}"
                    }
                }
            }
        }

        // === STAGE 4: Validate CloudFormation Template ===
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
                        
                        # Check if template exists
                        if [ -f "${TEMPLATE_FILE}" ]; then
                            echo "✅ Template found at ${TEMPLATE_FILE}"
                            cat ${TEMPLATE_FILE} | head -20
                        else
                            echo "❌ Template not found at ${TEMPLATE_FILE}!"
                            echo "Searching for template files..."
                            find . -name "*.yaml" -o -name "*.yml"
                            exit 1
                        fi
                        
                        # Validate template
                        aws cloudformation validate-template \
                            --template-body file://${TEMPLATE_FILE} \
                            --region ${AWS_DEFAULT_REGION}
                        
                        echo "✅ Template validation successful"
                    """
                }
            }
        }

        // === STAGE 5: Check if Stack Already Exists ===
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
                                --region ${AWS_DEFAULT_REGION} \
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

        // === STAGE 6: Delete Old Stack (if exists) ===
        stage('Delete Existing Stack (if any)') {
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
                            --stack-name ${STACK_NAME} \
                            --region ${AWS_DEFAULT_REGION}
                        """
                        
                        echo "⏳ Waiting for stack deletion to complete..."
                        sh """
                            aws cloudformation wait stack-delete-complete \
                            --stack-name ${STACK_NAME} \
                            --region ${AWS_DEFAULT_REGION}
                        """
                        echo "✅ Stack deleted successfully"
                    }
                }
            }
        }

        // === STAGE 7: Deploy Stack (Ubuntu) ===
        stage('Deploy Ubuntu EC2 Stack') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    script {
                        echo "🚀 Deploying Ubuntu EC2 stack: ${STACK_NAME}"
                        echo "🌍 Region: ${AWS_DEFAULT_REGION} (Mumbai)"
                        echo "🖼️ Ubuntu AMI: ${env.UBUNTU_AMI_ID}"
                        echo "💻 Instance Type: ${INSTANCE_TYPE}"
                        echo "🏷️ Environment: ${ENVIRONMENT}"
                        
                        def deployCmd = """
                            aws cloudformation deploy \
                            --template-file ${TEMPLATE_FILE} \
                            --stack-name ${STACK_NAME} \
                            --region ${AWS_DEFAULT_REGION} \
                            --parameter-overrides \
                                KeyName=${KEY_NAME} \
                                InstanceType=${INSTANCE_TYPE} \
                                AMIId=${env.UBUNTU_AMI_ID} \
                                Environment=${ENVIRONMENT} \
                                SSHLocation=0.0.0.0/0 \
                            --capabilities CAPABILITY_IAM \
                            --no-fail-on-empty-changeset
                        """
                        
                        echo "📋 Deployment command:"
                        echo "${deployCmd}"
                        
                        sh deployCmd
                        echo "✅ Stack deployment initiated successfully"
                    }
                }
            }
        }

        // === STAGE 8: Wait for Stack Creation ===
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
                            --stack-name ${STACK_NAME} \
                            --region ${AWS_DEFAULT_REGION}
                        """
                        // ↑ Pauses until stack is ready
                        echo "✅ Stack creation completed successfully"
                    }
                }
            }
        }

        // === STAGE 9: Get Stack Outputs ===
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
                                --region ${AWS_DEFAULT_REGION} \
                                --query "Stacks[0].Outputs" \
                                --output json
                            """,
                            returnStdout: true
                        )
                        
                        def outputsJson = readJSON text: outputs
                        
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
                            if (output.OutputKey == "SSHCommand") {
                                env.SSH_COMMAND = output.OutputValue
                            }
                            if (output.OutputKey == "WebURL") {
                                env.WEB_URL = output.OutputValue
                            }
                        }
                        
                        echo "=========================================="
                        echo "📋 STACK DEPLOYMENT DETAILS"
                        echo "=========================================="
                        outputsJson.each { output ->
                            echo "${output.OutputKey}: ${output.OutputValue}"
                        }
                        echo "=========================================="
                    }
                }
            }
        }

        // === STAGE 10: Test Web Server ===
        stage('Test Ubuntu Web Server') {
            steps {
                script {
                    echo "🌐 Testing web server..."
                    
                    // Wait a bit for Apache to fully start
                    sh "sleep 10"
                    
                    // Test web server
                    def response = sh(
                        script: """
                            curl -s -o /dev/null -w "%{http_code}" \
                            http://${env.EC2_PUBLIC_IP} || echo "000"
                        """,
                        returnStdout: true
                    ).trim()
                    
                    if (response == "200") {
                        echo "✅ Web server is responding with HTTP 200 OK!"
                        echo "🌐 Web page available at: http://${env.EC2_PUBLIC_IP}"
                        
                        // Get the actual web page content
                        def content = sh(
                            script: """
                                curl -s http://${env.EC2_PUBLIC_IP} || echo "Failed to get content"
                            """,
                            returnStdout: true
                        )
                        echo "📄 Web page content:"
                        echo "${content}"
                    } else {
                        echo "⚠️ Web server returned HTTP ${response}"
                        echo "Check if Apache started correctly"
                    }
                }
            }
        }

        // === STAGE 11: Deployment Summary ===
        stage('Deployment Summary') {
            steps {
                script {
                    echo "=========================================="
                    echo "✅ DEPLOYMENT SUCCESSFUL!"
                    echo "=========================================="
                    echo "🖥️ EC2 Instance Details:"
                    echo "  - OS: Ubuntu 22.04 LTS"
                    echo "  - Region: Mumbai (ap-south-1)"
                    echo "  - Public IP: ${env.EC2_PUBLIC_IP}"
                    echo "  - Public DNS: ${env.EC2_PUBLIC_DNS}"
                    echo "  - Instance ID: ${env.EC2_INSTANCE_ID}"
                    echo "  - Instance Type: ${INSTANCE_TYPE}"
                    echo ""
                    echo "🔑 SSH Access (Ubuntu):"
                    echo "  ssh -i ${KEY_NAME}.pem ubuntu@${env.EC2_PUBLIC_IP}"
                    echo "  ⚠️ NOTE: Ubuntu uses 'ubuntu' user, NOT 'ec2-user'!"
                    echo ""
                    echo "🌐 Web Access:"
                    echo "  http://${env.EC2_PUBLIC_IP}"
                    echo ""
                    echo "📊 Stack Management:"
                    echo "  Stack Name: ${STACK_NAME}"
                    echo "  Delete Stack: aws cloudformation delete-stack --stack-name ${STACK_NAME}"
                    echo "=========================================="
                    
                    // Save deployment info for later use
                    writeFile file: 'deployment-info.txt', text: """
                        DEPLOYMENT INFO
                        ===============
                        Stack Name: ${STACK_NAME}
                        Public IP: ${env.EC2_PUBLIC_IP}
                        SSH Command: ssh -i ${KEY_NAME}.pem ubuntu@${env.EC2_PUBLIC_IP}
                        Web URL: http://${env.EC2_PUBLIC_IP}
                        Environment: ${ENVIRONMENT}
                        Region: ${AWS_DEFAULT_REGION}
                        AMI: ${env.UBUNTU_AMI_ID}
                        Timestamp: ${new Date()}
                    """
                    
                    archiveArtifacts artifacts: 'deployment-info.txt'
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
            echo "Region: ${AWS_DEFAULT_REGION} (Mumbai)"
            echo ""
            echo "🌐 Access your Ubuntu server:"
            echo "  Web: http://${env.EC2_PUBLIC_IP}"
            echo "  SSH: ssh -i ${KEY_NAME}.pem ubuntu@${env.EC2_PUBLIC_IP}"
            echo "=========================================="
            
            // Optional: Send email notification
            emailext(
                subject: "✅ Ubuntu EC2 Deployment Successful - ${STACK_NAME}",
                body: """
                    Deployment of Ubuntu EC2 instance completed successfully!
                    
                    Stack Name: ${STACK_NAME}
                    Environment: ${ENVIRONMENT}
                    Region: ap-south-1 (Mumbai)
                    OS: Ubuntu 22.04 LTS
                    Instance Type: ${INSTANCE_TYPE}
                    
                    Access Details:
                    - Web URL: http://${env.EC2_PUBLIC_IP}
                    - SSH: ssh -i ${KEY_NAME}.pem ubuntu@${env.EC2_PUBLIC_IP}
                    
                    Stack Management:
                    - Delete: aws cloudformation delete-stack --stack-name ${STACK_NAME}
                    
                    Deployment Time: ${new Date()}
                """,
                to: 'team@example.com',
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
        
        // What to do when pipeline fails
        failure {
            echo "=========================================="
            echo "❌ DEPLOYMENT FAILED!"
            echo "=========================================="
            echo "Stack: ${STACK_NAME}"
            echo "Environment: ${ENVIRONMENT}"
            echo "Region: ${AWS_DEFAULT_REGION}"
            echo "=========================================="
            
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
                            --region ${AWS_DEFAULT_REGION} \
                            --max-items 10 \
                            --query "StackEvents[?ResourceStatus=='CREATE_FAILED' || ResourceStatus=='UPDATE_FAILED' || ResourceStatus=='ROLLBACK_COMPLETE']" \
                            --output table
                        """,
                        returnStdout: true
                    )
                    
                    echo "📋 Recent stack events:"
                    echo "${events}"
                    
                    // Get detailed error
                    def errorDetails = sh(
                        script: """
                            aws cloudformation describe-stack-events \
                            --stack-name ${STACK_NAME} \
                            --region ${AWS_DEFAULT_REGION} \
                            --max-items 5 \
                            --query "StackEvents[?ResourceStatusReason].[ResourceStatusReason]" \
                            --output text 2>/dev/null || echo 'No detailed error found'
                        """,
                        returnStdout: true
                    )
                    
                    echo "🔍 Error details:"
                    echo "${errorDetails}"
                }
            }
            
            echo "=========================================="
            echo "💡 Troubleshooting tips:"
            echo "1. Check AWS Console → CloudFormation → ${STACK_NAME}"
            echo "2. Verify your KeyPair '${KEY_NAME}' exists in Mumbai region"
            echo "3. Check if the Ubuntu AMI is available in ap-south-1"
            echo "4. Verify your AWS credentials have required permissions"
            echo "=========================================="
            
            // Send failure notification
            emailext(
                subject: "❌ Ubuntu EC2 Deployment Failed - ${STACK_NAME}",
                body: """
                    Deployment of Ubuntu EC2 instance FAILED!
                    
                    Stack Name: ${STACK_NAME}
                    Environment: ${ENVIRONMENT}
                    Region: ap-south-1 (Mumbai)
                    
                    Check AWS Console for details.
                    Deployment Time: ${new Date()}
                """,
                to: 'team@example.com'
            )
        }
        
        // Always run this
        always {
            echo "=========================================="
            echo "Pipeline execution completed"
            echo "Timestamp: ${new Date()}"
            echo "Build Number: ${env.BUILD_NUMBER}"
            echo "Job Name: ${env.JOB_NAME}"
            echo "=========================================="
            
            // Clean up workspace if needed
            // cleanWs()
        }
        
        // Clean up after build
        cleanup {
            echo "🧹 Cleaning up workspace..."
            // Delete temporary files
            sh '''
                rm -f deployment-info.txt
                rm -f *.log
            '''
        }
    }
}
