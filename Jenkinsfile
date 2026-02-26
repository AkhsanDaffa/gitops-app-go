pipeline {
    agent any

    environment {
        // Nama image docker yang akan dibuat
        IMAGE_NAME = "jawaracode/myapp"
        // Menggunakan Build Number dari Jenkins sebagai tag versi (v1, v2, v3, dst)
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
    }

    stages {
        // stage('1. Checkout Code') {
        //     steps {
        //         echo "Mengambil kode dari GitHub..."
        //         git branch: 'main', url: 'https://github.com/AkhsanDaffa/gitops-app-go.git'
        //     }
        // }

        stage('2. Unit Test (Golang)') {
            steps {
                echo "Menjalankan testing..."
                // sh '''
                // cat <<EOF > Dockerfile.test
                // FROM golang:1.22-alpine
                // WORKDIR /app
                // COPY . .
                // RUN go test -v ./...
                // EOF
                // docker build -t test-runner -f Dockerfile.test .
                // '''            
                
                sh '''
                cat <<EOF > Dockerfile.test
                FROM golang:1.22-alpine
                WORKDIR /app
                COPY . .
                RUN go test -v ./...
                EOF
                
                docker build -t test-runner -f Dockerfile.test .
                '''}
        }

        stage('3. Build Docker Image') {
            steps {
                echo "Membangun Docker Image versi ${IMAGE_TAG}..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
            }
        }

        stage('4. Push to Docker Hub') {
            when {
                branch 'main'
            }
            steps {
                echo "Mengirim image ke Docker Hub..."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                    sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${IMAGE_NAME}:latest"
                }
            }
        }
        
        stage('5. Update GitOps Manifest') {
            when {
                branch 'main'
            }
            steps {
                echo "Mengupdate versi image di repo manifest..."
                // Memanggil credential GitHub yang baru dibuat
                withCredentials([usernamePassword(credentialsId: 'github-token', passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
                    sh '''
                    # 0. Bersihkan sisa-sisa kegagalan sebelumnya (JIKA ADA)
                    rm -rf /tmp/gitops-manifest
                    
                    # 1. Clone repo manifest ke folder sementara (/tmp) agar tidak bentrok
                    git clone https://${GIT_USER}:${GIT_PASS}@github.com/AkhsanDaffa/gitops-manifest.git /tmp/gitops-manifest
                    cd /tmp/gitops-manifest
                    
                    # 2. Setup identitas Git (Jenkins yang melakukan commit)
                    git config user.email "jenkins@ci-server.local"
                    git config user.name "Jenkins Automation"
                    
                    # 3. Ganti tag versi di file deployment.yaml menggunakan 'sed'
                    sed -i "s|image: jawaracode/myapp:.*|image: jawaracode/myapp:${IMAGE_TAG}|g" deployment.yaml
                    
                    # 4. Commit dan Push ke GitHub
                    git add deployment.yaml
                    git commit -m "Deploy: Update image to ${IMAGE_TAG}"
                    git push https://${GIT_USER}:${GIT_PASS}@github.com/AkhsanDaffa/gitops-manifest.git main
                    
                    # 5. Bersihkan folder sementara
                    rm -rf /tmp/gitops-manifest
                    '''
                }
            }
        }
        
    }
    
    post {
        always {
            echo "Membersihkan workspace & logout docker..."
            sh "docker logout"
            cleanWs()
        }
        success {
            echo "✅ CI Pipeline BERHASIL! Image ${IMAGE_TAG} sudah ada di Docker Hub."
        }
        failure {
            echo "❌ CI Pipeline GAGAL. Silakan cek log."
        }
    }
}