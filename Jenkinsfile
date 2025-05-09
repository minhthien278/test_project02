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

    options {
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
    }

    stages {
        stage("Checkout") {
            steps {
                checkout scm
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

        stage('Test') {
            when {
                expression {
                    return env.CHANGED_SERVICES != null && env.CHANGED_SERVICES.trim()
                }
            }
            steps {
                script {
                    def services = env.CHANGED_SERVICES.split(',')
                    for (service in services) {
                        echo "Testing: ${service}"
                        sh "mvn clean verify -pl ${service}"
                    }   
                }
            }
        }

        stage('Build') {
            when {
                expression {
                    return env.CHANGED_SERVICES != null && env.CHANGED_SERVICES.trim()
                }
            }
            steps {
                script {
                    def services = env.CHANGED_SERVICES.split(',')
                    for (service in services) {
                        echo "Building: ${service}"
                        sh "mvn -pl ${service} -am package -DskipTests"
                    }  
                }
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([string(credentialsId: 'docker-hub-credentitals', variable: 'DOCKER_PASS')]) {
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
                    def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    def services = env.CHANGED_SERVICES.split(',')

                    for (service in services) {
                        def imageName = "vuden/${service}:${commitId}"
                        echo "🚀 Building and pushing image for ${service} with tag ${commitId}"
                        sh """ mvn clean install -pl ${service} -Dmaven.test.skip=true -P buildDocker 
                            -Ddocker.image.prefix=${env.DOCKER_USER} -Ddocker.image.tag=${commitId} -Dcontainer.build.extraarg=\"--push\""
                        """
                    }
                }
            }
        }
    }
}
//test