pipeline {
    agent any

    parameters {
        string(name: 'KEY_NAME', defaultValue: 'my-key-pair', description: 'EC2 KeyPair for SSH access')
        string(name: 'IMAGE_ID_FRONTEND', defaultValue: 'ami-0c55b159cbfafe1f0', description: 'AMI ID for Front-End EC2 instances')
        string(name: 'IMAGE_ID_BACKEND', defaultValue: 'ami-0c55b159cbfafe1f0', description: 'AMI ID for Back-End EC2 instances')
        string(name: 'TRUSTED_SSH_CIDR', defaultValue: '0.0.0.0/0', description: 'CIDR block for SSH access to EC2 instances')
    }

    environment {
        STACK_NAME = 'ProdEnvironmentStack' // Customize as needed
        TEMPLATE_FILE = 'cloudformation/prod-environment.yml' // Path to your template
        REGION = 'us-east-1' // AWS Region
        VPC_CIDR = '10.0.0.0/16'
        PUBLIC_SUBNET1_CIDR = '10.0.1.0/24'
        PUBLIC_SUBNET2_CIDR = '10.0.2.0/24'
    }

    stages {
        stage('Checkout') {
            steps {
                // Replace with your repository details
                git branch: 'main',
                    url: 'https://github.com/your-repo/cloudformation-templates.git'
            }
        }

        stage('Deploy CloudFormation Stack') {
            steps {
                script {
                    // Optionally, fetch the latest AMI IDs using AWS SSM Parameter Store
                    /*
                    IMAGE_ID_FRONTEND = sh (
                        script: "aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 --region ${REGION} --query 'Parameters[0].Value' --output text",
                        returnStdout: true
                    ).trim()

                    IMAGE_ID_BACKEND = sh (
                        script: "aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 --region ${REGION} --query 'Parameters[0].Value' --output text",
                        returnStdout: true
                    ).trim()
                    */

                    // Deploy or update the CloudFormation stack
                    sh """
                        aws cloudformation deploy \
                            --template-file ${TEMPLATE_FILE} \
                            --stack-name ${STACK_NAME} \
                            --region ${REGION} \
                            --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
                            --parameter-overrides \
                                KeyName=${params.KEY_NAME} \
                                ImageIdFrontEnd=${params.IMAGE_ID_FRONTEND} \
                                ImageIdBackEnd=${params.IMAGE_ID_BACKEND} \
                                VpcCidr=${VPC_CIDR} \
                                PublicSubnet1Cidr=${PUBLIC_SUBNET1_CIDR} \
                                PublicSubnet2Cidr=${PUBLIC_SUBNET2_CIDR} \
                                TrustedSSHCIDR=${params.TRUSTED_SSH_CIDR}
                    """

                    // Wait for stack to be fully deployed
                    sh """
                        aws cloudformation wait stack-update-complete --stack-name ${STACK_NAME} --region ${REGION}
                    """

                    // Retrieve stack outputs and set them as environment variables
                    def stackOutputs = sh (
                        script: "aws cloudformation describe-stacks --stack-name ${STACK_NAME} --region ${REGION} --query 'Stacks[0].Outputs' --output json",
                        returnStdout: true
                    ).trim()

                    def outputs = readJSON text: stackOutputs

                    // Extract specific outputs
                    def loadBalancerDNS = outputs.find { it.OutputKey == 'LoadBalancerDNS' }?.OutputValue
                    def frontEndASGName = outputs.find { it.OutputKey == 'FrontEndAutoScalingGroupName' }?.OutputValue
                    def backEndASGName = outputs.find { it.OutputKey == 'BackEndAutoScalingGroupName' }?.OutputValue

                    // Set environment variables for subsequent stages
                    env.LOAD_BALANCER_DNS = loadBalancerDNS
                    env.FRONT_END_ASG_NAME = frontEndASGName
                    env.BACK_END_ASG_NAME = backEndASGName

                    // (Optional) Echo the outputs
                    echo "Load Balancer DNS: ${env.LOAD_BALANCER_DNS}"
                    echo "Front-End ASG Name: ${env.FRONT_END_ASG_NAME}"
                    echo "Back-End ASG Name: ${env.BACK_END_ASG_NAME}"
                }
            }
        }

        stage('Post-Deployment Steps') {
            steps {
                // Example: Notify stakeholders, run integration tests, etc.
                echo "Deployment completed successfully!"
                echo "ALB DNS: ${env.LOAD_BALANCER_DNS}"
                echo "Front-End ASG: ${env.FRONT_END_ASG_NAME}"
                echo "Back-End ASG: ${env.BACK_END_ASG_NAME}"
                // You can add more steps here as needed
            }
        }
    }

    post {
        success {
            echo 'CloudFormation stack deployed successfully!'
            // Optional: Send success notifications (e.g., Slack, Email)
        }
        failure {
            echo 'CloudFormation stack deployment failed.'
            // Optional: Send failure notifications
        }
    }
}
