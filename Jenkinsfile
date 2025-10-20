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

        stage('Deploy to Minikube') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=${KUBECONFIG}
                        kubectl config use-context ${MINIKUBE_CONTEXT}

                        # Appliquer les fichiers YAML depuis la racine
                        kubectl apply -f deployment.yaml
                        kubectl apply -f service.yaml

                        # Vérifier le déploiement
                        kubectl get pods -o wide
                        kubectl get svc student-app-service
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished"
        }
    }
}
