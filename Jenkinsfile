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
        KUBE_CONFIG_PATH = "$HOME/.kube/config"
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
                        kubectl config use-context minikube
                        kubectl set image deployment/student-app student-app=${DOCKER_IMAGE}:latest --record || true
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        kubectl get pods -o wide
                    """
                }
            }
        }
    }
}
