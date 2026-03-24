pipeline {
    agent any

    environment {
        IMAGE_NAME = 'grunt0341/java-app:latest'
        SONARQUBE_SERVER = 'http://sonar:9000'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build with Java 17') {
            steps {
                sh "docker run --rm --volumes-from jenkins -w ${WORKSPACE} maven:3.9.6-eclipse-temurin-17 mvn -B clean package -DskipTests -Dcheckstyle.skip=true -Dspring-javaformat.skip=true"
            }
        }

        stage('Run Tests with Java 11') {
            steps {
                sh "docker run --rm --volumes-from jenkins -w ${WORKSPACE} maven:3.9.6-eclipse-temurin-11 mvn -B test -Dcheckstyle.skip=true -Dspring-javaformat.skip=true -Dsurefire.failIfNoSpecifiedTests=false"
            }
        }

        stage('Analyze Code Quality') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "docker run --rm --volumes-from jenkins --network ci_network -w ${WORKSPACE} maven:3.9.6-eclipse-temurin-17 mvn -B sonar:sonar -Dsonar.projectKey=spring-petclinic -Dsonar.host.url=http://sonar:9000 -Dsonar.token=${SONAR_AUTH_TOKEN} -Dcheckstyle.skip=true -Dspring-javaformat.skip=true"
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
                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity HIGH,CRITICAL ${IMAGE_NAME}"
            }
        }

        stage('Push Docker Image') {
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
