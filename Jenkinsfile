pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        JAVA_HOME = tool 'JDK-21'
        PATH = "${JAVA_HOME}\\bin;${env.PATH}"
        MAVEN_OPTS = '-Dmaven.repo.local=.m2/repository'
        IMAGE_TAG = "${env.BRANCH_NAME ?: 'dev'}-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Config Server') {
            steps {
                bat 'mvnw.cmd -B -pl spring-petclinic-config-server -am clean package -DskipTests'
            }
        }

        stage('Build Discovery Server') {
            steps {
                bat 'mvnw.cmd -B -pl spring-petclinic-discovery-server -am clean package -DskipTests'
            }
        }

        stage('Start MySQL') {
            steps {
                bat '''
                    docker compose down --remove-orphans || exit 0
                    docker rm -f mysql-server 2>NUL || exit 0
                    docker compose up -d mysql
                '''
            }
        }

        stage('Build API Gateway + Admin Server') {
            parallel {
                stage('Build API Gateway') {
                    steps {
                        bat 'mvnw.cmd -B -pl spring-petclinic-api-gateway -am clean package -DskipTests'
                    }
                }
                stage('Build Admin Server') {
                    steps {
                        bat 'mvnw.cmd -B -pl spring-petclinic-admin-server -am clean package -DskipTests'
                    }
                }
            }
        }

        stage('Build Business Services') {
            parallel {
                stage('Build Customers Service') {
                    steps {
                        bat 'mvnw.cmd -B -pl spring-petclinic-customers-service -am clean package -DskipTests'
                    }
                }
                stage('Build Vets Service') {
                    steps {
                        bat 'mvnw.cmd -B -pl spring-petclinic-vets-service -am clean package -DskipTests'
                    }
                }
                stage('Build Visits Service') {
                    steps {
                        bat 'mvnw.cmd -B -pl spring-petclinic-visits-service -am clean package -DskipTests'
                    }
                }
                stage('Build GenAI Service') {
                    steps {
                        bat 'mvnw.cmd -B -pl spring-petclinic-genai-service -am clean package -DskipTests'
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                bat 'mvnw.cmd -B clean install -P buildDocker -DskipTests'
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
                    bat '''
                        echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin

                        for %%I in (
                            spring-petclinic-admin-server
                            spring-petclinic-config-server
                            spring-petclinic-customers-service
                            spring-petclinic-discovery-server
                            spring-petclinic-genai-service
                            spring-petclinic-vets-service
                            spring-petclinic-visits-service
                            spring-petclinic-api-gateway
                        ) do (
                            docker tag %%I:v1 %DOCKER_USER%/%%I:%IMAGE_TAG%
                            docker push %DOCKER_USER%/%%I:%IMAGE_TAG%
                            docker tag %%I:v1 %DOCKER_USER%/%%I:latest
                            docker push %DOCKER_USER%/%%I:latest
                        )
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            when {
                branch 'main'
            }
            steps {
                bat '''
                    docker compose down --remove-orphans || exit 0
                    docker compose up -d --remove-orphans
                    docker compose ps
                '''
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
        success {
            echo 'Jenkins CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'Jenkins CI/CD pipeline failed. Check the Maven and Docker logs.'
        }
    }
}

