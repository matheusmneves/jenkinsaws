pipeline {
    agent any

    environment {
        AWS_DOCKER_REGISTRY = '612634926349.dkr.ecr.us-east-2.amazonaws.com'
        APP_NAME = 'my_new_image'
        AWS_DEFAULT_REGION = 'us-east-2'
        S3_BUCKET_NAME = 'matheusnevesjenkinsreact'
    }
    
    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:22-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # list all files
                    ls -la
                    node --version
                    npm --version
                    # npm install
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'node:22-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    test -f build/index.html
                    npm test
                '''
            }
        }
        
        // Novo estágio para fazer upload no S3
        stage('Deploy to S3') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    reuseNode true
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-s3-credentials', 
                                             passwordVariable: 'AWS_SECRET_ACCESS_KEY', 
                                             usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        # Sincronizar os arquivos de build com o bucket S3
                        aws s3 sync build/ s3://${S3_BUCKET_NAME}/ --delete
                        
                        # Configurando o bucket para hospedagem web (opcional)
                        aws s3 website s3://${S3_BUCKET_NAME}/ --index-document index.html --error-document index.html
                        
                        echo "Site disponível em: http://${S3_BUCKET_NAME}.s3-website-${AWS_DEFAULT_REGION}.amazonaws.com/"
                    '''
                }
            }
        }

        stage('Build My Docker Image'){
            agent{
                docker{
                    image 'amazon/aws-cli'
                    reuseNode true
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint=""'
                }
            }
            steps{
                withCredentials([usernamePassword(credentialsId: 'myNewUser', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        amazon-linux-extras install docker
                        docker build -t $AWS_DOCKER_REGISTRY/$APP_NAME .
                        aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_DOCKER_REGISTRY
                        docker push $AWS_DOCKER_REGISTRY/$APP_NAME:latest
                    '''
                }
            }
        }

        stage('Deploy to AWS'){
            agent{
                docker{
                    image 'amazon/aws-cli'
                    reuseNode true
                    args '-u root --entrypoint=""'
                }
            }
            steps{
                withCredentials([usernamePassword(credentialsId: 'myNewUser', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        yum install jq -y
                        LATEST_TD_REVISION=$(aws ecs register-task-definition --cli-input-json file://aws/task-definition.json | jq '.taskDefinition.revision')
                        aws ecs update-service --cluster my-new-react-app-Cluster-Prod --service my-new-react-app-Service-Prod --task-definition MyNewReactApp-TaskDefinition-Prod:$LATEST_TD_REVISION
                    '''
                }
            }
        }
    }
}