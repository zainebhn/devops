pipeline {
    agent any

    tools {
        maven 'Maven 3.6.3'
        jdk 'Java 17'
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

        stage('Run') {
            steps {
                dir('student-management') {
                    sh 'mvn spring-boot:run &'
                }
            }
        }
    }
}
