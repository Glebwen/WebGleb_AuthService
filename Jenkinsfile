pipeline {
    agent any
    
    environment {
        PATH = "C:\\Program Files\\Docker\\Docker\\resources\\bin;${env.PATH}"
        REPO = 'https://github.com/Glebwen/DevOPSLab2.git'
        DOCKER_BUILDKIT = '1'
        REGISTRY = 'localhost:5000'
    }
    
    triggers {
        pollSCM('H/1 * * * *') 
    }
    
    stages {
        stage('Checkout') {
            steps { 
                checkout scm 
            }
        }
        
        stage('Docker Info') {
            steps {
                bat 'docker version'
            }
        }
        
        stage('Test (Build & Pytest)') {
            steps {
                bat 'docker build -t devopslab2-test .'
                bat 'docker run --rm devopslab2-test python -m pytest'
            }
        }
        
        stage('Push to Local Registry') {
            when { branch 'main' }
            steps {
                bat """
                    docker ps -a --filter "name=local-registry" --format "{{.Names}}" | findstr "local-registry" >nul
                    if errorlevel 1 (
                        echo "Запускаем локальный Docker Registry..."
                        docker run -d -p 5000:5000 --restart=always --name local-registry registry:2
                        ping 127.0.0.1 -n 4 > nul
                    ) else (
                        echo "Локальный Registry уже работает."
                    )

                    docker compose build
                    
                    docker push %REGISTRY%/devopslab2-backend:latest
                    docker push %REGISTRY%/devopslab2-frontend:latest
                """
            }
        }
        
        stage('Deploy') {
            when { branch 'main' }
            steps {
                bat """
                    if not exist C:\\deploy\\DevOPSLab2 mkdir C:\\deploy\\DevOPSLab2
                    if exist C:\\deploy\\DevOPSLab2\\.git (
                        cd /d C:\\deploy\\DevOPSLab2 & git pull
                    ) else (
                        git clone %REPO% C:\\deploy\\DevOPSLab2
                    )
                    cd /d C:\\deploy\\DevOPSLab2
                    
                    if not exist db.sqlite3 type nul > db.sqlite3
                    
                    docker compose down --remove-orphans || exit /b 0
                    
                    docker rm -f devopslab2-backend-1 devopslab2-frontend-1 2>nul || exit /b 0
                    
                    docker compose pull
                    
                    docker compose up -d
                    
                    ping 127.0.0.1 -n 6 > nul
              
                    docker compose exec -T backend python manage.py migrate
                """
            }
        }

        stage('Deploy to K8s') {
            when { branch 'main' }
            steps {
                bat """
                    docker compose build
                    minikube image load localhost:5000/devopslab2-backend:latest
                    minikube image load localhost:5000/devopslab2-frontend:latest
                    kubectl apply -f k8s/
                    kubectl rollout restart deployment/backend-deployment
                    kubectl rollout restart deployment/frontend-deployment
                """
            }
        }
    }
    
    post {
        success {
            echo '=== Pipeline SUCCESS ==='
            echo 'Сайт доступен по адресу: http://localhost:8081'
            echo 'Локальный Registry доступен по адресу: http://localhost:5000/v2/_catalog'
        }
        failure {
            echo '=== Pipeline FAILED ==='
        }
    }
}