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

        stage('Debug Dockerfile') {
            steps {
               sh 'pwd && ls -la Dockerfile && head -3 Dockerfile'
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
                    script {
                        sh 'docker rm -f mysql-test-container || true'
                        sh '''
                            docker run -d --rm --name mysql-test-container \
                                -e MYSQL_ROOT_PASSWORD=root \
                                -e MYSQL_DATABASE=studentdb \
                                -p 3307:3306 mysql:8.0
                        '''
                        sh '''
                            until docker exec mysql-test-container mysqladmin ping -h localhost -uroot -proot --silent; do
                                sleep 2
                            done
                        '''
                        sh '''
                            mvn test \
                                -Dspring.profiles.active=test \
                                -Dspring.datasource.url=jdbc:mysql://localhost:3307/studentdb?createDatabaseIfNotExist=true \
                                -Dspring.datasource.username=root \
                                -Dspring.datasource.password=root
                        '''
                        sh 'docker rm -f mysql-test-container || true'
                    }
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
                    sh '''
                        minikube status || (echo "❌ Minikube not running" && exit 1)
                        kubectl config use-context minikube
                        kubectl create namespace devops --dry-run=client -o yaml | kubectl apply -f -

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
