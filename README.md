# python-flask-project

**End-to-End CI/CD Pipeline with Python, Jenkins, ArgoCD, and EKS**
This guide provides a complete walkthrough of setting up a CI/CD pipeline for a Python application using Jenkins for CI, ArgoCD for CD, and deploying to Amazon EKS.

Prerequisites
**•	AWS account with EKS cluster
•	kubectl configured to access your EKS cluster
•	Jenkins installed and configured
•	ArgoCD installed in your EKS cluster
•	GitHub account
•	Docker Hub account (or other container registry)**

Step 1: Create a Sample Python Application
First, let's create a simple Python Flask application.
1.1 Project Structure
python-flask-app/
├── app/
│   ├── __init__.py
│   ├── main.py
│   └── templates/
│       └── index.html
├── tests/
│   └── test_app.py
├── requirements.txt
├── Dockerfile
├── Jenkinsfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── README.md

1.2 **Sample Application Code
Create a new GitHub repository (e.g., python-flask-app) and add these files:**
app/main.py:
python
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/')
def hello():
    return render_template('index.html')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
app/templates/index.html:
html
<!DOCTYPE html>
<html>
<head>
    <title>Python Flask App</title>
</head>
<body>
    <h1>Welcome to our Python Flask App!</h1>
    <p>Version: {{ version }}</p>
</body>
</html>
requirements.txt:
flask==2.0.1
pytest==6.2.5
pytest-cov==3.0.0
tests/test_app.py:
python
import pytest
from app.main import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_hello(client):
    response = client.get('/')
    assert response.status_code == 200
    assert b'Welcome' in response.data
Dockerfile:
dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV FLASK_APP=app/main.py
ENV FLASK_ENV=production

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app.main:app"]
Step 2: Kubernetes Manifests
Create Kubernetes deployment and service files in the k8s directory:
k8s/deployment.yaml:
yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-flask-app
  labels:
    app: python-flask-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: python-flask-app
  template:
    metadata:
      labels:
        app: python-flask-app
    spec:
      containers:
      - name: python-flask-app
        image: <your-dockerhub-username>/python-flask-app:latest
        ports:
        - containerPort: 5000
        env:
        - name: VERSION
          value: "1.0.0"
k8s/service.yaml:
yaml
apiVersion: v1
kind: Service
metadata:
  name: python-flask-app
spec:
  selector:
    app: python-flask-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: LoadBalancer
Step 3: Jenkins Pipeline Setup
3.1 Install Required Jenkins Plugins
•	Docker Pipeline
•	Kubernetes
•	Git
•	Blue Ocean (optional)
3.2 Create Jenkinsfile
Jenkinsfile:
groovy
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
3.3 Configure Jenkins Credentials
1.	Add Docker Hub credentials in Jenkins:
o	Kind: Username with password
o	ID: dockerhub-credentials
o	Username: your-dockerhub-username
o	Password: your-dockerhub-password
2.	Add kubeconfig file as a credential in Jenkins
3.	Add GitHub credentials if using private repo
Step 4: ArgoCD Setup and Configuration
4.1 Install ArgoCD in EKS Cluster
bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
4.2 Access ArgoCD UI
bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc -n argocd argocd-server
Get the initial admin password:
bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
4.3 Create ArgoCD Application
Create an application YAML file (argocd-app.yaml):
yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: python-flask-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-username/python-flask-app.git'
    path: k8s
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
Apply it:
bash
kubectl apply -f argocd-app.yaml
Step 5: Trigger the Pipeline
1.	Make a change to your Python application code and push to GitHub
2.	Jenkins will automatically trigger the pipeline (if webhook is configured)
3.	The pipeline will:
o	Run tests
o	Build and push Docker image
o	Update Kubernetes manifests
4.	ArgoCD will detect the changes in the Git repo and deploy the new version to EKS
Step 6: Verify Deployment
Check your application:
bash
kubectl get svc python-flask-app
Access the external IP in your browser to see the Flask application running.
Additional Improvements
1.	Helm Charts: Instead of raw Kubernetes manifests, use Helm charts for more flexibility
2.	Secrets Management: Use AWS Secrets Manager or HashiCorp Vault for secrets
3.	Monitoring: Add Prometheus and Grafana for monitoring
4.	Logging: Set up EFK (Elasticsearch, Fluentd, Kibana) stack for logging
5.	Security Scanning: Add security scanning in the pipeline (Trivy, Snyk)
6.	Blue-Green Deployment: Implement more advanced deployment strategies
Sample GitHub Repositories
Here are some sample repositories you can reference:
1.	Simple Python Flask App
2.	Python with Jenkins Pipeline
3.	ArgoCD Examples
4.	EKS Deployment Examples
Conclusion
This end-to-end pipeline demonstrates how to:
1.	Develop a Python application
2.	Set up CI with Jenkins
3.	Build and push Docker images
4.	Deploy to EKS using GitOps with ArgoCD
5.	Automate the entire process from code commit to production deployment
The pipeline ensures that every code change goes through testing, builds a deployable artifact, and automatically updates the production environment while maintaining the benefits of GitOps practices.

