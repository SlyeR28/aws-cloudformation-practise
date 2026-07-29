pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = "us-east-1"
        STACK_NAME = "dev-ec2-stack"
        ENVIRONMENT = "dev"
        KEY_NAME = "jenkins-ec2"
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('AWS Login Test') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {
                    sh '''
                    aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Get Latest AMI') {
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

                    }
                }
            }
        }

        stage('Validate Template') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {

                    sh """
                    aws cloudformation validate-template \
                    --template-body file://cloudformationTempletes/ec2.yaml
                    """

                }
            }
        }

        stage('Deploy Stack') {
            steps {
                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials'
                    ]
                ]) {

                    sh """
                    aws cloudformation deploy \
                    --template-file cloudformationTempletes/ec2.yaml \
                    --stack-name ${STACK_NAME} \
                    --parameter-overrides \
                    KeyName=${KEY_NAME} \
                    InstanceType=t2.micro \
                    AMIId=${AMI_ID} \
                    --capabilities CAPABILITY_IAM \
                    --no-fail-on-empty-changeset
                    """

                }
            }
        }

    }
}
