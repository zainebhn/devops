
pipeline {
    agent any

    tools {
        maven 'MAVEN/usr/share/maven'
        jdk 'JDK17'
    }

    environment {
        MAVEN_OPTS   = "-Dmaven.test.skip=true"
        DOCKER_IMAGE = "zainebheni/student-management" // ton image DockerHub
        SONARQUBE_ENV = "sonarqube" // nom du serveur SonarQube configuré dans Jenkins
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
                                  -Dsonar.login=$SONAR_TOKEN
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
                            docker build -t $DOCKER_IMAGE:latest .
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                            docker push $DOCKER_IMAGE:latest
                        """
                    }
                }
            }
        }

        stage('Run App in Docker') {
            steps {
                script {
                    sh """
                        docker rm -f student-app || true
                        docker run -d -p 8080:8080 --name student-app $DOCKER_IMAGE:latest
                    """
                }
            }
        }
    }
}


