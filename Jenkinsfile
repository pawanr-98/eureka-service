pipeline {
    agent none 
    environment {
        BRANCH = "${env.BRANCH_NAME}"
        SERVICE_NAME = "eureka"
        DOCKER_IMAGE = "pdock855/dr-front:${SERVICE_NAME}-${env.BRANCH_NAME}-${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            agent { label 'built-in' }
            steps {
                cleanWs()
                git branch: "${BRANCH}", credentialsId: 'github', url: 'https://github.com/pawanr-98/eureka-service.git'
            }
        }

        stage('Build & Push to Docker Hub') {
            agent { label 'built-in' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker build -t $DOCKER_IMAGE .
                    docker push $DOCKER_IMAGE
                    docker logout
                    """
                }
            }
        }

        stage('Eureka Deployment') {
            agent { label 'master-node' }
            steps {
                script {
                    def namespace = ''
                    if (env.BRANCH_NAME == 'main') {
                        namespace = 'ums-prod'
                    } else if (env.BRANCH_NAME == 'uat') {
                        namespace = 'ums-uat'
                    } else {
                        error("Unsupported branch ${env.BRANCH_NAME}")
                    }

                    withCredentials([string(credentialsId: 'ums-cluster', variable: 'CLUSTER_CONTEXT')]) {
                        sh """
                        kubectl config use-context $CLUSTER_CONTEXT
                        kubectl apply -f deployment.yaml -n ${namespace}
                        kubectl apply -f service.yaml -n ${namespace}
                        kubectl set image deployment/ums-eureka ums-eureka=$DOCKER_IMAGE -n ${namespace}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployed $SERVICE_NAME to namespace: ${env.BRANCH_NAME}"
        }
        failure {
            echo "Deployment failed for $SERVICE_NAME to ${env.BRANCH_NAME}"
        }
    }
}
