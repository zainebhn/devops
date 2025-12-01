pipeline {
    agent any

    environment {
        MAVEN_OPTS = "-Xmx1024m"
        DOCKER_IMAGE = "zainebheni/student-management"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
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
                    sh 'mvn clean compile -DskipTests'
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

        stage('Package') {
            steps {
                dir('student-management') {
                    sh 'mvn package -DskipTests'
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
                        docker login -u \$DOCKERHUB_USR -p \$DOCKERHUB_PSW
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                script {
                    // Vérifie que Minikube est démarré
                    sh '''
                        minikube status || (echo "❌ Minikube not running" && exit 1)
                        kubectl cluster-info
                    '''

                    // Applique les manifestes (depuis ton repo)
                    sh '''
                        kubectl apply -f https://raw.githubusercontent.com/zainebhn/devops/main/mysql-deployment.yaml -n devops
                        kubectl wait --for=condition=ready pod -l app=mysql -n devops --timeout=300s

                        kubectl apply -f https://raw.githubusercontent.com/zainebhn/devops/main/deployment.yaml -n devops
                        kubectl apply -f https://raw.githubusercontent.com/zainebhn/devops/main/service.yaml -n devops
                        kubectl set image deployment/student-app student-app=${DOCKER_IMAGE}:${DOCKER_TAG} -n devops
                        kubectl rollout status deployment/student-app -n devops --timeout=300s
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl get pods -n devops
                    kubectl get svc -n devops
                '''
            }
        }
    }

    post {
        success {
            script {
                def ip = sh(script: 'minikube ip', returnStdout: true).trim()
                echo "✅ Application disponible à : http://${ip}:30081"
            }
        }
        failure {
            echo '❌ Pipeline échoué'
        }
        always {
            cleanWs()
        }
    }
}
