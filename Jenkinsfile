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

        // Novo estágio para fazer upload para o S3
        stage('Deploy to S3') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-s3-credentials', 
                                       passwordVariable: 'AWS_SECRET_ACCESS_KEY', 
                                       usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        # Instalar ferramentas necessárias
                        apt-get update || true
                        apt-get install -y curl openssl || true
                        
                        # Data atual para a assinatura
                        DATE=$(date -u +"%a, %d %b %Y %H:%M:%S GMT")
                        
                        # Função para fazer upload
                        upload_to_s3() {
                            local FILE_PATH=$1
                            local CONTENT_TYPE=$2
                            
                            # Criar string para assinar
                            local STRING_TO_SIGN="PUT\\n\\n${CONTENT_TYPE}\\n${DATE}\\n/${S3_BUCKET_NAME}/${FILE_PATH}"
                            
                            # Gerar assinatura
                            local SIGNATURE=$(echo -en "${STRING_TO_SIGN}" | openssl sha1 -hmac "${AWS_SECRET_ACCESS_KEY}" -binary | base64)
                            
                            # Upload do arquivo
                            curl -X PUT -T "build/${FILE_PATH}" \\
                                 -H "Host: ${S3_BUCKET_NAME}.s3.amazonaws.com" \\
                                 -H "Date: ${DATE}" \\
                                 -H "Content-Type: ${CONTENT_TYPE}" \\
                                 -H "Authorization: AWS ${AWS_ACCESS_KEY_ID}:${SIGNATURE}" \\
                                 "https://${S3_BUCKET_NAME}.s3.amazonaws.com/${FILE_PATH}"
                        }
                        
                        # Encontrar todos os arquivos na pasta build e fazer upload
                        cd build
                        find . -type f | while read -r file; do
                            # Remover o ./ do início do caminho
                            filepath=$(echo "$file" | sed 's|^./||')
                            
                            # Determinar o content-type
                            if [[ $filepath == *.html ]]; then
                                content_type="text/html"
                            elif [[ $filepath == *.css ]]; then
                                content_type="text/css"
                            elif [[ $filepath == *.js ]]; then
                                content_type="application/javascript"
                            elif [[ $filepath == *.png ]]; then
                                content_type="image/png"
                            elif [[ $filepath == *.jpg ]] || [[ $filepath == *.jpeg ]]; then
                                content_type="image/jpeg"
                            elif [[ $filepath == *.svg ]]; then
                                content_type="image/svg+xml"
                            elif [[ $filepath == *.json ]]; then
                                content_type="application/json"
                            elif [[ $filepath == *.ico ]]; then
                                content_type="image/x-icon"
                            else
                                content_type="application/octet-stream"
                            fi
                            
                            echo "Fazendo upload de $filepath ($content_type)..."
                            upload_to_s3 "$filepath" "$content_type"
                        done
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