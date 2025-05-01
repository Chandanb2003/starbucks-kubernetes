pipeline {
    agent any
    tools {
        jdk 'jdk'
        nodejs 'node17'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME = 'starbucks'
        NAMESPACE = 'default'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/Aseemakram19/starbucks-kubernetes.git'
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=starbucks \
                    -Dsonar.projectKey=starbucks '''
                }
            }
        }

        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {   
                        sh "docker build -t starbucks ."
                        sh "docker tag starbucks chandan669/starbucks:latest"
                        sh "docker push chandan669/starbucks:latest"
                    }
                }
            }
        }

        stage("TRIVY") {
            steps {
                sh "trivy image chandan669/starbucks:latest > trivyimage.txt"
            }
        }

        stage('App Deploy to Docker container') {
            steps {
                sh 'docker run -d --name starbucks -p 3000:3000 chandan669/starbucks:latest'
            }
        }

        stage('Install Kubernetes Cluster') {
            steps {
                sh '''
                echo "📦 Installing Kubernetes using kubeadm..."
                sudo apt-get update
                sudo apt-get install -y apt-transport-https curl
                curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
                echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
                sudo apt-get update
                sudo apt-get install -y kubelet kubeadm kubectl
                sudo kubeadm init --pod-network-cidr=192.168.0.0/16 || true
                mkdir -p $HOME/.kube
                sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
                sudo chown $(id -u):$(id -g) $HOME/.kube/config
                kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                echo "🚀 Deploying to Kubernetes..."
                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml
                '''
            }
        }

        stage('Show Application Port & Access Info') {
            steps {
                sh '''
                echo "🌐 Fetching app port..."
                NODE_PORT=$(kubectl get svc ${APP_NAME}-svc -n ${NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}')
                PUBLIC_IP=$(curl -s ifconfig.me)
                echo "✅ App is running at: http://$PUBLIC_IP:$NODE_PORT"
                '''
            }
        }
    }
}
