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
                        # Export kubeconfig pour Jenkins
                        export KUBECONFIG=${KUBECONFIG}
                        kubectl config use-context minikube

                        # Créer deployment.yaml à la volée
                        cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: student-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: student-app
  template:
    metadata:
      labels:
        app: student-app
    spec:
      containers:
      - name: student-app
        image: ${DOCKER_IMAGE}:latest
        ports:
        - containerPort: 8080
EOF

                        # Créer service.yaml à la volée
                        cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: student-app-service
spec:
  type: NodePort
  selector:
    app: student-app
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30082
EOF

                        # Appliquer les fichiers Kubernetes
                        kubectl apply -f deployment.yaml
                        kubectl apply -f service.yaml

                        # Vérifier les pods
                        kubectl get pods -o wide
                    """
                }
            }
        }
    }
}
