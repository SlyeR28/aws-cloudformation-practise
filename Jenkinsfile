pipeline {
    agent any
    
    environment {
        // AWS Credentials from Jenkins
        AWS_ACCESS_KEY_ID = credentials('aws-credentials-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-credentials-secret-key')
        
        AWS_REGION = 'us-east-1'
        ENVIRONMENT = 'dev'
        STACK_NAME = 'dev-ec2-stack'
        KEY_NAME = 'jenkins-ec2'       
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Deploy EC2') {
            steps {
                sh '''
                    AMI_ID=$(aws ec2 describe-images \
                        --owners amazon \
                        --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
                        --query "Images | sort_by(@, &CreationDate) | [-1].ImageId" \
                        --output text)
                    
                    echo "Using AMI: $AMI_ID"
                    
                    aws cloudformation deploy \
                        --template-file cloudformationTempletes/ec2.yaml \
                        --stack-name ${STACK_NAME} \
                        --parameter-overrides \
                            EnvironmentName=${ENVIRONMENT} \
                            KeyName=${KEY_NAME} \
                            AMIId=$AMI_ID \
                        --region ${AWS_REGION} \
                        --capabilities CAPABILITY_IAM \
                        --no-fail-on-empty-changeset
                    
                    echo "✅ Stack deployed successfully!"
                '''
            }
        }
    }
}
