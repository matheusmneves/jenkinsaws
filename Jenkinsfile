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
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_DEFAULT_REGION=${AWS_REGION}
                        
                        # Instalar AWS CLI (versão simplificada)
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip -q awscliv2.zip
                        ./aws/install -i /tmp/aws-cli -b /tmp/aws-bin
                        export PATH=/tmp/aws-bin:$PATH
                        
                        # Verificar instalação do AWS CLI
                        aws --version
                        
                        # Sincronizar conteúdo da pasta build para o bucket
                        aws s3 sync build/ s3://${S3_BUCKET_NAME}/ --delete
                        
                        echo "Upload completo!"
                    '''
                }
            }
        }
    }
}