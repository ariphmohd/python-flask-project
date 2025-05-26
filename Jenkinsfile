pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        KUBECONFIG = credentials('kubeconfig')
        GIT_REPO = 'https://github.com/your-username/python-flask-app.git'
        DOCKER_IMAGE = 'your-dockerhub-username/python-flask-app'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: env.GIT_REPO
            }
        }
        
        stage('Test') {
            steps {
                sh 'python -m pytest tests/ --cov=app --cov-report=xml'
            }
            post {
                always {
                    junit '**/test-results.xml'
                    cobertura coberturaReportFile: '**/coverage.xml'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build(env.DOCKER_IMAGE)
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('', env.DOCKERHUB_CREDENTIALS) {
                        docker.image(env.DOCKER_IMAGE).push('latest')
                    }
                }
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                sh """
                    sed -i "s|<your-dockerhub-username>/python-flask-app:latest|${env.DOCKER_IMAGE}:latest|g" k8s/deployment.yaml
                    git config --global user.email "jenkins@example.com"
                    git config --global user.name "Jenkins"
                    git add k8s/deployment.yaml
                    git commit -m "Update image version in deployment [skip ci]"
                    git push https://${env.GIT_CREDENTIALS}@github.com/your-username/python-flask-app.git HEAD:main
                """
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
