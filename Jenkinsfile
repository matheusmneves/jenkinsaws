pipeline {
    agent any
    
    environment {
        S3_BUCKET_NAME = 'matheusnevesjenkinsreact'
        AWS_REGION = 'us-east-2'
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
                    npm -v
                    node -v
                '''
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    npm install
                    npm run build
                    # Verificar o conteúdo do build
                    ls -la build/
                    cat build/index.html
                '''
            }
        }
        
        stage('Upload to S3') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-s3-credentials', 
                                               passwordVariable: 'AWS_SECRET_ACCESS_KEY', 
                                               usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        # Installar aws cli via curl
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        apt-get update || true
                        apt-get install -y unzip || true
                        unzip -o awscliv2.zip
                        ./aws/install || true
                        
                        # Configurar AWS CLI
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_DEFAULT_REGION=${AWS_REGION}
                        
                        # Tentar usar AWS CLI para upload
                        echo "Tentando upload via AWS CLI..."
                        aws --version
                        
                        # Sincronizar toda a pasta build com o bucket, tornando tudo público
                        aws s3 sync build/ s3://${S3_BUCKET_NAME}/ --acl public-read --delete || true
                        
                        # Método alternativo: usando curl
                        if [ $? -ne 0 ]; then
                            echo "AWS CLI não funcionou, usando curl..."
                            # Data atual para a assinatura
                            DATE=$(date -u +"%a, %d %b %Y %H:%M:%S GMT")
                            
                            # Função para upload com curl
                            upload_file() {
                                FILE_PATH=$1
                                CONTENT_TYPE=$2
                                
                                STRING_TO_SIGN="PUT\\n\\n${CONTENT_TYPE}\\n${DATE}\\n/${S3_BUCKET_NAME}/${FILE_PATH}"
                                SIGNATURE=$(echo -en "${STRING_TO_SIGN}" | openssl sha1 -hmac "${AWS_SECRET_ACCESS_KEY}" -binary | base64)
                                
                                echo "Uploading ${FILE_PATH} to s3://${S3_BUCKET_NAME}/${FILE_PATH}"
                                curl -v -X PUT -T "build/${FILE_PATH}" \\
                                    -H "Host: ${S3_BUCKET_NAME}.s3.${AWS_REGION}.amazonaws.com" \\
                                    -H "Date: ${DATE}" \\
                                    -H "Content-Type: ${CONTENT_TYPE}" \\
                                    -H "Authorization: AWS ${AWS_ACCESS_KEY_ID}:${SIGNATURE}" \\
                                    "https://${S3_BUCKET_NAME}.s3.${AWS_REGION}.amazonaws.com/${FILE_PATH}"
                                
                                echo "Upload status for ${FILE_PATH}: $?"
                            }
                            
                            # Upload index.html
                            upload_file "index.html" "text/html"
                            
                            # Upload JS files
                            find build -name "*.js" | while read file; do
                                rel_path=$(echo "$file" | sed 's|build/||')
                                upload_file "$rel_path" "application/javascript"
                            done
                            
                            # Upload CSS files
                            find build -name "*.css" | while read file; do
                                rel_path=$(echo "$file" | sed 's|build/||')
                                upload_file "$rel_path" "text/css"
                            done
                            
                            # Upload other files
                            find build -name "*.png" -o -name "*.ico" -o -name "*.json" | while read file; do
                                rel_path=$(echo "$file" | sed 's|build/||')
                                
                                if [[ $file == *.png ]]; then
                                    content_type="image/png"
                                elif [[ $file == *.ico ]]; then
                                    content_type="image/x-icon"
                                elif [[ $file == *.json ]]; then
                                    content_type="application/json"
                                fi
                                
                                upload_file "$rel_path" "$content_type"
                            done
                        fi
                        
                        echo "Upload completo!"
                        echo "Site disponível em: http://${S3_BUCKET_NAME}.s3-website.${AWS_REGION}.amazonaws.com/"
                    '''
                }
            }
        }
    }
}