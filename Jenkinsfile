pipeline {
    agent any

    tools {
        maven 'MAVEN/usr/share/maven'
        jdk 'JDK17'

    }
    environment{
        MAVEN_OPTS = "-Dmaven.test.skip=true" 
        DOCKERHUB_CREDENTIALS = credentials('docker-hub')
        IMAGE_NAME = "zainebheni/student-management"
        IMAGE_TAG = "latest"
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

        stage('Test') {
            steps {
                dir('student-management') {
                    sh 'mvn test'
                }
            }
        }

         stage('Docker Build & Push') {
            steps {
                script {
                    dir('student-management') {
                        sh """
                        docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy (Run Container)') {
            steps {
                sh """
                docker stop student-management || true
                docker rm student-management || true
                docker run -d -p 8080:8080 --name student-management ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }
}
