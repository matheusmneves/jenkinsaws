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
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_DEFAULT_REGION=${AWS_REGION}
                        
                        # Instalar AWS CLI (com opção 'overwrite')
                        rm -rf aws-cli awscliv2.zip aws
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip -o awscliv2.zip
                        ./aws/install -i /tmp/aws-cli -b /tmp/aws-bin
                        
                        # Adicionar ao PATH
                        export PATH=/tmp/aws-bin:$PATH
                        
                        # Verificar a instalação
                        /tmp/aws-bin/aws --version
                        
                        # Upload direto para o bucket
                        /tmp/aws-bin/aws s3 sync build/ s3://${S3_BUCKET_NAME}/ --delete
                        
                        echo "Upload completo!"
                        echo "Site disponível em: http://${S3_BUCKET_NAME}.s3-website.${AWS_REGION}.amazonaws.com/"
                    '''
                }
            }
        }
    }
}
