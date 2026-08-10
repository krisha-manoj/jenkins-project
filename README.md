Spring Boot CI/CD Pipeline with Jenkins, SonarQube, Docker & ArgoCD

This is a Spring Boot based Java web application built using Maven with a complete CI/CD pipeline. The application follows MVC architecture where the controller returns a page with title and message attributes to the view.

Prerequisites
AWS EC2 instance (Ubuntu, t3.medium or larger, 20GB EBS)
Java 17
Docker
Jenkins
SonarQube
kubectl & eksctl
Helm
GitHub account with Personal Access Token
Docker Hub account
AWS EKS cluster
Execute the Application Locally
Clone the repo

git clone https://github.com/krisha-manoj/jenkinsproject.git
cd jenkinsproject/java-maven-sonar-argocd-helm-k8s/spring-boot-app
Build the artifacts

mvn clean package
Run locally (Java 17 needed)

java -jar target/spring-boot-web.jar
Access at: http://localhost:8080

The Docker Way

docker build -t krishasuman123/ultimate-cicd:v1 .
docker run -d -p 8010:8080 -t krishasuman123/ultimate-cicd:v1
Access at: http://<ip-address>:8010

Install Jenkins on EC2

sudo apt update
sudo apt install openjdk-17-jdk net-tools -y
wget https://pkg.jenkins.io/debian-stable/binary/jenkins_2.462.3_all.deb
sudo dpkg -i jenkins_2.462.3_all.deb
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Access at: http://<ip-address>:8080

Note: Add Jenkins user to Docker group:


sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
Install SonarQube on EC2

sudo apt update && sudo apt install unzip -y
sudo adduser sonarqube
sudo apt install openjdk-17-jdk -y
sudo update-alternatives --set java /usr/lib/jvm/java-17-openjdk-amd64/bin/java
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip
unzip sonarqube-10.4.1.88267.zip
sudo mv sonarqube-10.4.1.88267 /opt/sonarqube
sudo chown -R sonarqube:sonarqube /opt/sonarqube
sudo chmod -R 775 /opt/sonarqube
su - sonarqube -c "/opt/sonarqube/bin/linux-x86-64/sonar.sh start"
Access at: http://<ip-address>:9000 (default login: admin / admin)

Important Notes:

SonarQube 10.4 requires Java 17 — Java 21 will NOT work
Ensure at least 15% free disk space — Elasticsearch will fail otherwise
If disk is full, grow the EBS volume:

sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
Install Docker on EC2

sudo apt-get install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
Build Custom Docker Agent (Java 17 + Maven + Docker)
The default abhishekf5/maven-abhishek-docker-agent:v1 has Java 11 which is incompatible with SonarQube 10.4. Build a custom agent:


cd ~
cat > Dockerfile << 'EOF'
FROM maven:3.9.6-eclipse-temurin-17
RUN apt-get update && apt-get install -y docker.io && rm -rf /var/lib/apt/lists/*
EOF
docker build -t krishasuman123/maven-docker-agent:v2 .
docker push krishasuman123/maven-docker-agent:v2
Configure Jenkins Credentials
Go to Manage Jenkins → Credentials → Global → Add Credentials:

ID	Kind	Value
github-creds	Username with password	GitHub username + PAT
github	Secret text	GitHub PAT
docker-cred	Username with password	Docker Hub username + password
sonarqube	Secret text	SonarQube token (My Account → Security → Generate)
Jenkins Pipeline (Final Working Jenkinsfile)

pipeline {
  agent {
    docker {
      image 'krishasuman123/maven-docker-agent:v2'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/krisha-manoj/Jenkinsproj.git', credentialsId: 'github-creds'
      }
    }
    stage('Build and Test') {
      steps {
        sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn clean package'
      }
    }
    stage('Static Code Analysis') {
      environment {
        SONAR_URL = "http://172.31.25.181:9000"
      }
      steps {
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
          sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
        }
      }
    }
    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "krishasuman123/ultimate-cicd:${BUILD_NUMBER}"
        REGISTRY_CREDENTIALS = credentials('docker-cred')
      }
      steps {
        script {
            sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
            def dockerImage = docker.image("${DOCKER_IMAGE}")
            docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                dockerImage.push()
            }
        }
      }
    }
    stage('Update Deployment File') {
        environment {
            GIT_REPO_NAME = "Jenkinsproj"
            GIT_USER_NAME = "krisha-manoj"
        }
        steps {
            withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                sh '''
                    git config --global --add safe.directory /var/lib/jenkins/workspace/jenkinscicd
                    git config --global user.email "krishasuman123@gmail.com"
                    git config --global user.name "Krisha Manoj"
                    BUILD_NUMBER=${BUILD_NUMBER}
                    sed -i "s/ultimate-cicd:.*/ultimate-cicd:${BUILD_NUMBER}/g" java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
                    git add java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
                    git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                    git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                '''
            }
        }
    }
  }
}
Key fixes applied:

Used custom Docker agent with Java 17 (maven-docker-agent:v2)
Used EC2 private IP for SonarQube (172.31.25.181) — Docker containers can't reach public IP
Added git config --global --add safe.directory — Docker runs as root, workspace owned by Jenkins
Used sed -i "s/ultimate-cicd:.*/ultimate-cicd:${BUILD_NUMBER}/g" — works on every build, not just the first
Deploy with ArgoCD on EKS
Set up EKS Cluster

aws eks update-kubeconfig --name jenkinsproj --region us-west-2
Fix cluster access (required for every new EKS cluster):


# Allow VPC traffic to cluster
aws ec2 authorize-security-group-ingress --region us-west-2 \
  --group-id <cluster-sg> --protocol all --cidr 172.31.0.0/16

# Add IAM user access
aws eks create-access-entry --region us-west-2 --cluster-name jenkinsproj \
  --principal-arn arn:aws:iam::307946644949:user/oregonuser --type STANDARD
aws eks associate-access-policy --region us-west-2 --cluster-name jenkinsproj \
  --principal-arn arn:aws:iam::307946644949:user/oregonuser \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
Install ArgoCD

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
Create ArgoCD Application

cat > ~/app.yml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: spring-boot-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/krisha-manoj/Jenkinsproj.git
    targetRevision: main
    path: java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
EOF
kubectl apply -f ~/app.yml
Access ArgoCD UI

pkill -f "port-forward"
kubectl port-forward svc/argocd-server -n argocd 8443:443 --address 0.0.0.0 &
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
Open: https://<ec2-public-ip>:8443 (open port 8443 in EC2 security group)

Login: admin / password from above command
