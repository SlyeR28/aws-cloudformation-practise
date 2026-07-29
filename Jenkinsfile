pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = "us-east-1"
        STACK_NAME = "dev-ec2-stack"
        ENVIRONMENT = "dev"
        KEY_NAME = "jenkins-ec2"
    }

    parameters {
        string(
            name: 'TEMPLATE_FILE',
            defaultValue: 'cloudformationTempletes/ec2.yaml',
            description: 'CloudFormation template path'
        )
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Verify Environment') {
            steps {
                sh '''
                    echo "Checking AWS CLI..."
                    aws --version

                    echo "Checking Git..."
                    git --version

                    echo "Workspace:"
                    pwd

                    ls -R
                '''
            }
        }

        stage('Get Latest Amazon Linux 2 AMI') {
            steps {
                script {
                    env.AMI_ID = sh(
                        script: """
                        aws ec2 describe-images \
                        --owners amazon \
                        --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
                        --query "Images | sort_by(@,&CreationDate)[-1].ImageId" \
                        --output text \
                        --region ${AWS_DEFAULT_REGION}
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Latest AMI: ${env.AMI_ID}"
                }
            }
        }

        stage('Validate CloudFormation Template') {
            steps {
                sh """
                aws cloudformation validate-template \
                --template-body file://${params.TEMPLATE_FILE} \
                --region ${AWS_DEFAULT_REGION}
                """
            }
        }

        stage('Deploy Stack') {
            steps {
                sh """
                aws cloudformation deploy \
                  --template-file ${params.TEMPLATE_FILE} \
                  --stack-name ${STACK_NAME} \
                  --parameter-overrides \
                    EnvironmentName=${ENVIRONMENT} \
                    KeyName=${KEY_NAME} \
                    AMIId=${AMI_ID} \
                  --capabilities CAPABILITY_IAM \
                  --region ${AWS_DEFAULT_REGION} \
                  --no-fail-on-empty-changeset
                """
            }
        }

        stage('Describe Stack') {
            steps {
                sh """
                aws cloudformation describe-stacks \
                --stack-name ${STACK_NAME} \
                --region ${AWS_DEFAULT_REGION}
                """
            }
        }

        stage('Stack Outputs') {
            steps {
                sh """
                aws cloudformation describe-stacks \
                --stack-name ${STACK_NAME} \
                --query "Stacks[0].Outputs" \
                --output table \
                --region ${AWS_DEFAULT_REGION}
                """
            }
        }
    }

    post {

        success {
            echo "======================================="
            echo "CloudFormation Deployment Successful"
            echo "Stack Name : ${STACK_NAME}"
            echo "Region     : ${AWS_DEFAULT_REGION}"
            echo "======================================="
        }

        failure {
            echo "Deployment Failed"

            sh """
            aws cloudformation describe-stack-events \
            --stack-name ${STACK_NAME} \
            --max-items 10 \
            --region ${AWS_DEFAULT_REGION} || true
            """
        }

        always {
            cleanWs()
        }
    }
}