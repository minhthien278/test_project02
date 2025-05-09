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
        // Define missing environment variables
        DOCKER_REGISTRY = "vuden"  // Using your docker hub username from the original script
        CONTAINER_TAG = "\${BUILD_NUMBER}-\${GIT_COMMIT.substring(0,7)}"  // Using build number and git commit as tag
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
                    def testResults = [:]
                    def hasFailures = false
                    
                    // Run tests for all services, even if some fail
                    for (service in services) {
                        echo "Testing: ${service}"
                        try {
                            sh "./mvnw clean verify -pl ${service}"
                            testResults[service] = "SUCCESS"
                        } catch (Exception e) {
                            testResults[service] = "FAILED"
                            hasFailures = true
                            echo "Test failed for ${service}: ${e.message}"
                        }
                    }
                    
                    // Report test results summary
                    echo "Test results summary:"
                    testResults.each { service, result ->
                        echo "${service}: ${result}"
                    }
                    
                    // Fail the build if any test failed
                    if (hasFailures) {
                        error "Some tests failed. Check the logs for details."
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
                        sh "./mvnw -pl ${service} -am package -DskipTests"
                    }  
                }
            }
        }

        stage ('Login docker hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }
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
                        def imageName = "${DOCKER_REGISTRY}/${service}:${commitId}"
                        echo "🚀 Building and pushing image for ${service} with tag ${commitId}"
                        
                        // Using the defined environment variables
                        sh """
                            ./mvnw clean install -pl ${service} -Dmaven.test.skip=true -P buildDocker \
                            -Ddocker.image.prefix=${DOCKER_REGISTRY} \
                            -Ddocker.image.tag=${commitId} \
                            -Dcontainer.build.extraarg="--push"
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Clean up after build
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed! Check the logs for details."
        }
    }
}