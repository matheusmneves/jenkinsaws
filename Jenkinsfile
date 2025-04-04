pipeline {
    agent any
    
    environment {
        S3_BUCKET_NAME = 'matheusnevesjenkinsreact' // Substitua pelo nome real do seu bucket
        AWS_REGION = 'us-east-2'             // Mantenha a mesma região
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                sh '''
                    apt-get update || true
                    apt-get install -y nodejs npm curl openssl python3-pip || true
                    npm -v
                    node -v
                    pip3 install awscli || true
                    aws --version || true
                '''
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    npm install
                    npm run build
                '''
            }
        }
        
        stage('Upload to S3') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-s3-credentials', 
                                               passwordVariable: 'AWS_SECRET_ACCESS_KEY', 
                                               usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        # Data atual para a assinatura
                        DATE=$(date -u +"%a, %d %b %Y %H:%M:%S GMT")
                        
                        # Upload usando curl
                        upload_file() {
                            FILE_PATH=$1
                            CONTENT_TYPE=$2
                            
                            STRING_TO_SIGN="PUT\\n\\n${CONTENT_TYPE}\\n${DATE}\\n/${S3_BUCKET_NAME}/${FILE_PATH}"
                            SIGNATURE=$(echo -en "${STRING_TO_SIGN}" | openssl sha1 -hmac "${AWS_SECRET_ACCESS_KEY}" -binary | base64)
                            
                            curl -X PUT -T "build/${FILE_PATH}" \\
                                -H "Host: ${S3_BUCKET_NAME}.s3.amazonaws.com" \\
                                -H "Date: ${DATE}" \\
                                -H "Content-Type: ${CONTENT_TYPE}" \\
                                -H "Authorization: AWS ${AWS_ACCESS_KEY_ID}:${SIGNATURE}" \\
                                "https://${S3_BUCKET_NAME}.s3.amazonaws.com/${FILE_PATH}"
                                
                            echo "Uploaded: ${FILE_PATH}"
                        }
                        
                        # Verificar se a pasta build existe
                        if [ ! -d "build" ]; then
                            echo "ERROR: pasta build não encontrada!"
                            exit 1
                        fi
                        
                        # Listar arquivos na pasta build
                        echo "Arquivos na pasta build:"
                        ls -la build/
                        
                        # Fazer upload de arquivos principais manualmente
                        echo "Fazendo upload de arquivos principais..."
                        upload_file "index.html" "text/html"
                        
                        # Localizar e fazer upload de arquivos js
                        echo "Fazendo upload de arquivos JS..."
                        find build -name "*.js" | while read file; do
                            rel_path=$(echo "$file" | sed 's|build/||')
                            upload_file "$rel_path" "application/javascript"
                        done
                        
                        # Localizar e fazer upload de arquivos css
                        echo "Fazendo upload de arquivos CSS..."
                        find build -name "*.css" | while read file; do
                            rel_path=$(echo "$file" | sed 's|build/||')
                            upload_file "$rel_path" "text/css"
                        done
                        
                        echo "Upload completo!"
                        echo "Site disponível em: http://${S3_BUCKET_NAME}.s3-website-${AWS_REGION}.amazonaws.com/"
                    '''
                }
            }
        }
    }
}