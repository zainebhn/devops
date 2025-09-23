pipeline {
    agent any

    tools {
        maven 'MAVEN/usr/share/maven'
        jdk 'JDK17'

    }
    environment{
        MAVEN_OPTS = "-Dmaven.test.skip=true" 
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
