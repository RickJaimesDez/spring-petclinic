pipeline {
    agent any
    environment {
        SONARQUBE_SERVER = 'http://sonar:9000'
        IMAGE_NAME = 'grunt0341/java-app:latest'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build with Java 17') {
            steps {
                script {
                    sh 'docker run --rm --volumes-from jenkins -w ${WORKSPACE} --user root alpine sh -c "rm -rf target"'
                    docker.image('maven:3.9-eclipse-temurin-17').inside("--network ci_network --volumes-from jenkins") {
                        sh '''
                        export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
                        mvn -B clean package -DskipTests \
                            -Dspring-javaformat.skip=true \
                            -Dnative-maven-plugin.skip=true \
                            -Denforcer.skip=true
                        '''
                    }
                }
            }
        }
        stage('Run Tests with Java 11') {
            steps {
                script {
                    docker.image('maven:3.9-eclipse-temurin-11').inside("--network ci_network --volumes-from jenkins") {
                        sh '''
                        mvn -B test -DskipTests \
                            -Dspring-javaformat.skip=true \
                            -Dnative-maven-plugin.skip=true \
                            -Denforcer.skip=true
                        '''
                    }
                }
            }
        }
        stage('Verify Java 8 Environment') {
            steps {
                script {
                    docker.image('eclipse-temurin:8-jdk').inside("--network ci_network --volumes-from jenkins") {
                        sh '''
                        echo "=== Java 8 Environment Verification ==="
                        java -version
                        echo "Java 8 container successfully verified"
                        '''
                    }
                }
            }
        }
        stage('Static Code Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        docker.image('maven:3.9-eclipse-temurin-11').inside("--network ci_network --volumes-from jenkins") {
                            sh "mvn -B sonar:sonar -Dsonar.host.url=${SONARQUBE_SERVER} -Dsonar.java.source=1.8 -Dspring-javaformat.skip=true -Dnative-maven-plugin.skip=true -Denforcer.skip=true"
                        }
                    }
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }
        stage('Trivy Security Scan') {
            steps {
                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:0.61.0 image --severity HIGH,CRITICAL ${IMAGE_NAME}"
            }
        }
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push grunt0341/java-app:latest
                    '''
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh 'KUBECONFIG=/var/jenkins_home/.kube/config kubectl apply -f deployment.yaml --validate=false'
            }
        }
    }
}
