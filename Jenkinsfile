pipeline {
    agent any

    environment {
        SERVICES = """
            spring-petclinic-admin-server
            spring-petclinic-api-gateway
            spring-petclinic-config-server
            spring-petclinic-customers-service
            spring-petclinic-discovery-server
            spring-petclinic-genai-service
            spring-petclinic-vets-service
            spring-petclinic-visits-service
        """
        DOCKER_USER = 'vuden'
    }

    stages {
        stage("Checkout") {
            steps {
                checkout scm
            }
        }
        stage('Detect Release') {
            when { expression { return env.TAG_NAME } }
            steps {
                script {
                    echo "A new release found with tag ${env.TAG_NAME}"
                    env.CHANGED_SERVICES = env.SERVICES
                }
            }
        }
        stage('Detect Changes') {
            steps {
                script {
                    sh "git fetch origin main:refs/remotes/origin/main"
                    def changes = sh(script: "git diff --name-only origin/main HEAD", returnStdout: true).trim().split("\n")
                    echo "Changed files: ${changes}"

                    def allServices = SERVICES.split().collect { it.trim() }

                    def changedServices = allServices.findAll { service ->
                        changes.any { it.contains(service) }
                    }

                    if (changedServices.isEmpty()) {
                        echo "No service changes detected. Skipping build."
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    echo "Detected changed services: ${changedServices}"
                    env.CHANGED_SERVICES = changedServices.join(',')
                }
            }
        }

        stage('Docker Login') {
            when {
                expression {
                    return env.CHANGED_SERVICES != null && env.CHANGED_SERVICES.trim()
                }
            }
            steps {
                sh 'whoami'
                withCredentials([string(credentialsId: 'docker-credentials', variable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }


        stage('Build and push docker image') {
            when {
                expression {
                    return env.CHANGED_SERVICES != null && env.CHANGED_SERVICES.trim()
                }
            }
            steps {
                script {
                    def imageTag;
                    if (env.TAG_NAME) {
                        imageTag = env.TAG_NAME
                    } else {
                        imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    }
                    def services = env.CHANGED_SERVICES.split(',')

                    for (service in services) {
                        def imageName = "vuden/${service}:${commitId}"
                        echo "🚀 Building and pushing image for ${service} with tag ${commitId}"
                        // sh "./mvnw clean install -pl ${service} -P buildDocker -Ddocker.image.prefix=${env.DOCKER_USER} -Ddocker.image.tag=${imageName}"
                        //sh "cd ${service} && ../mvnw clean install -P BuilDocker "
                        sh "docker build -t ${DOCKER_USER}/${service}:${imageTag} -f ../docker/Dockerfile -- build-arg ARTIFACT_NAME=target/${service}-3.4.1 -- build-arg EXPOSE_PORT=8080" 
                        sh "docker push ${DOCKER_USER}/${service}:${imageTag}"
                    }
                }
            }
        }
    }
}
