pipeline {
    agent any
    environment {
        MAVEN_OPTS = "-Dmaven.test.skip=true"
        DOCKER_IMAGE = "zainebheni/student-management"
        SONARQUBE_ENV = "sonarqube"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
        MINIKUBE_CONTEXT = "minikube"
        NODE_PORT = "30081"
        APP_PORT = "8089"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'cd5fe940-75d1-4257-8b4c-d245de66f89c',
                    url: 'https://github.com/zainebhn/devops.git'
            }
        }

        stage('Build') {
            steps {
                dir('student-management') {
                    sh 'mvn clean install -DskipTests'
                }
            }
        }

        stage('Unit Tests') {
            steps {
                dir('student-management') {
                    sh 'mvn test'
                }
            }
            post {
                always {
                    junit 'student-management/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('sonarqube-token')
            }
            steps {
                dir('student-management') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=student-management \
                          -Dsonar.host.url=http://localhost:9000 \
                          -Dsonar.login=\${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Docker Build & Push') {
            environment {
                DOCKERHUB = credentials('docker-hub')
            }
            steps {
                dir('student-management') {
                    sh """
                        docker login -u \${DOCKERHUB_USR} -p \${DOCKERHUB_PSW}
                        docker build -t ${DOCKER_IMAGE}:latest .
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy Database to Minikube') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=${KUBECONFIG}
                        kubectl config use-context ${MINIKUBE_CONTEXT}

                        kubectl create namespace devops --dry-run=client -o yaml | kubectl apply -f -
                        kubectl apply -f https://raw.githubusercontent.com/zainebhn/devops/main/mysql-deployment.yaml -n devops
                        kubectl wait --for=condition=ready pod -l app=mysql -n devops --timeout=300s
                    """
                }
            }
        }

        stage('Deploy Application to Minikube') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=${KUBECONFIG}
                        kubectl config use-context ${MINIKUBE_CONTEXT}

                        kubectl apply -f https://raw.githubusercontent.com/zainebhn/devops/main/deployment.yaml -n devops
                        kubectl apply -f https://raw.githubusercontent.com/zainebhn/devops/main/service.yaml -n devops

                        kubectl wait --for=condition=ready pod -l app=student-app -n devops --timeout=300s

                        echo 'Pods:'
                        kubectl get pods -n devops -o wide
                        echo 'Services:'
                        kubectl get svc -n devops
                    """
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished'
        }
        success {
            script {
                sh '''
                    APP_IP=$(minikube ip)
                    echo "Application deployed successfully at: http://${APP_IP}:30081"
                '''
            }
        }
    }
}
