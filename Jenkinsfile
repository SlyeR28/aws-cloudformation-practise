cd ~/aws-cloudformation-practise

cat > Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    environment {
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
                    AMI_ID=$(aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" --query "Images | sort_by(@, &CreationDate) | [-1].ImageId" --output text)
                    echo "Using AMI: $AMI_ID"
                    aws cloudformation deploy --template-file cloudformationTempletes/ec2.yaml --stack-name dev-ec2-stack --parameter-overrides EnvironmentName=dev KeyName=jenkins-ec2 AMIId=$AMI_ID --region us-east-1 --capabilities CAPABILITY_IAM --no-fail-on-empty-changeset
                    echo "Stack deployed successfully!"
                '''
            }
        }
    }
}
EOF
