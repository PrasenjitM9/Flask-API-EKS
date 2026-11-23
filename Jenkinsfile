pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '123456789012'
        ECR_REPOSITORY = 'my-app-repo'
        ECS_CLUSTER = 'my-app-cluster'
        ECS_SERVICE = 'my-app-service'
        TASK_FAMILY = 'my-app-task'
        CONTAINER_NAME = 'my-app-container'
        IMAGE_TAG = "${BUILD_NUMBER}"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/my-app-repo:${BUILD_NUMBER}"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/your-user/your-repo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${ECR_REPOSITORY}:${IMAGE_TAG} .
                '''
            }
        }

        stage('AWS ECR Login') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                        aws ecr get-login-password \
                        --region ${AWS_REGION} |
                        docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                    '''
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    docker tag ${ECR_REPOSITORY}:${IMAGE_TAG} ${IMAGE_URI}

                    docker push ${IMAGE_URI}
                '''
            }
        }

        stage('Update ECS Task Definition') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                        aws ecs describe-task-definition \
                        --task-definition ${TASK_FAMILY} \
                        --region ${AWS_REGION} \
                        --query taskDefinition \
                        > task-definition.json

                        jq --arg IMAGE "${IMAGE_URI}" \
                           --arg CONTAINER "${CONTAINER_NAME}" \
                           '
                           .containerDefinitions |=
                           map(if .name == $CONTAINER
                               then .image = $IMAGE
                               else .
                               end)
                           |
                           del(
                               .taskDefinitionArn,
                               .revision,
                               .status,
                               .requiresAttributes,
                               .compatibilities,
                               .registeredAt,
                               .registeredBy,
                               .deregisteredAt
                           )
                           ' task-definition.json \
                           > new-task-definition.json

                        aws ecs register-task-definition \
                        --cli-input-json file://new-task-definition.json \
                        --region ${AWS_REGION} \
                        --query 'taskDefinition.taskDefinitionArn' \
                        --output text > task-revision.txt
                    '''
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {
                    sh '''
                        TASK_REVISION=$(cat task-revision.txt)

                        aws ecs update-service \
                        --cluster ${ECS_CLUSTER} \
                        --service ${ECS_SERVICE} \
                        --task-definition ${TASK_REVISION} \
                        --region ${AWS_REGION}

                        aws ecs wait services-stable \
                        --cluster ${ECS_CLUSTER} \
                        --services ${ECS_SERVICE} \
                        --region ${AWS_REGION}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Application successfully deployed to AWS ECS!'
        }

        failure {
            echo 'CI/CD Pipeline failed. Check Jenkins console output.'
        }

        always {
            sh 'docker image prune -f || true'
        }
    }
}
