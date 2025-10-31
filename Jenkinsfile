
pipeline {
    agent any

    tools {
        maven 'MAVEN/usr/share/maven'
        jdk 'JDK17'
    }

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
        }

        stage('SonarQube Analysis') {
            steps {
                dir('student-management') {
                    withSonarQubeEnv("${SONARQUBE_ENV}") {
                        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                            sh """
                                mvn sonar:sonar \
                                  -Dsonar.projectKey=student-management \
                                  -Dsonar.host.url=http://localhost:9000 \
                                  -Dsonar.login=\$SONAR_TOKEN
                            """
                        }
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                dir('student-management') {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            docker login -u '$DOCKER_USER' -p '$DOCKER_PASS'
                            docker build -t '$DOCKER_USER/student-management:latest' .
                            docker push '$DOCKER_USER/student-management:latest'
                        """
                    }
                }
            }
        }

        stage('Deploy Database to Minikube') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=${KUBECONFIG}
                        kubectl config use-context ${MINIKUBE_CONTEXT}
                        
                        # Create namespace if not exists
                        kubectl create namespace devops --dry-run=client -o yaml | kubectl apply -f -
                        
                        # Deploy MySQL first
                        kubectl apply --insecure-skip-tls-verify=true --validate=false -f https://raw.githubusercontent.com/zainebhn/devops/main/mysql-deployment.yaml -n devops
                        
                        # Wait for MySQL to be ready
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

                        # Deploy application
                        kubectl apply --insecure-skip-tls-verify=true --validate=false -f https://raw.githubusercontent.com/zainebhn/devops/main/deployment.yaml -n devops
                        kubectl apply --insecure-skip-tls-verify=true --validate=false -f https://raw.githubusercontent.com/zainebhn/devops/main/service.yaml -n devops

                        # Wait for application to be ready
                        kubectl wait --for=condition=ready pod -l app=student-app -n devops --timeout=300s

                        # Verification
                        echo 'Liste des pods:'
                        kubectl get pods -n devops -o wide

                        echo 'Liste des services:'
                        kubectl get svc -n devops

                        echo 'Application URL:'
                        minikube service student-app-service -n devops --url
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished"
        }
        success {
            script {
                sh """
                    export KUBECONFIG=${KUBECONFIG}
                    kubectl config use-context ${MINIKUBE_CONTEXT}
                    APP_URL=\$(minikube service student-app-service -n devops --url)
                    echo "Application deployed successfully at: \${APP_URL}"
                """
            }
        }
    }
}
