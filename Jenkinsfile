pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        JAVA_HOME = tool 'JDK-21'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        MAVEN_OPTS = '-Dmaven.repo.local=.m2/repository'
        REGISTRY = 'docker.io'
        IMAGE_TAG = "${env.BRANCH_NAME ?: 'dev'}-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh './mvnw -B clean verify'
            }
        }

        stage('Build Docker Images') {
            steps {
                sh './mvnw -B clean install -P buildDocker -DskipTests'
            }
        }

        stage('Publish Images to Registry') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        for image in \
                            spring-petclinic-admin-server \
                            spring-petclinic-config-server \
                            spring-petclinic-customers-service \
                            spring-petclinic-discovery-server \
                            spring-petclinic-genai-service \
                            spring-petclinic-vets-service \
                            spring-petclinic-visits-service \
                            spring-petclinic-api-gateway; do
                            docker tag ${image}:v1 "$DOCKER_USER/${image}:${IMAGE_TAG}"
                            docker push "$DOCKER_USER/${image}:${IMAGE_TAG}"
                            docker tag ${image}:v1 "$DOCKER_USER/${image}:latest"
                            docker push "$DOCKER_USER/${image}:latest"
                        done
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    docker compose down --remove-orphans || true
                    docker compose up -d --remove-orphans
                    docker compose ps
                '''
            }
        }
    }

    post {
        always {
            junit '**/target/surefire-reports/*.xml'
        }
        success {
            echo 'Jenkins CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'Jenkins CI/CD pipeline failed. Check the Maven and Docker logs.'
        }
    }
}
