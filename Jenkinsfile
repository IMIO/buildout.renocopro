@Library('jenkins-pipeline-scripts') _

pipeline {
    agent none
    triggers {
        pollSCM('*/3 * * * *')
    }
    options {
        // Keep the 50 most recent builds
        buildDiscarder(logRotator(numToKeepStr:'30'))
    }
    stages {
        stage('Build') {
            agent any
            steps {
                sh 'make docker-image'
            }
        }
        stage('Push image to registry') {
            agent any
            steps {
                pushImageToRegistry (
                    env.BUILD_ID,
                    'renocopro/mutual'
                )
            }
        }
        stage('Deploy to staging') {
            agent any
            when {
                expression {
                    currentBuild.result == null || currentBuild.result == 'SUCCESS'
                }
            }
            steps {
                echo "mco shell run 'docker pull docker-staging.imio.be/renocopro/mutual:$BUILD_ID' -I /^staging.imio.be/"
                echo "mco shell run 'systemctl restart renocopro.service' -I /^staging.imio.be/"
            }
        }
        stage('Deploy to prod ?') {
            agent none
            steps {
                timeout(time: 24, unit: 'HOURS') {
                    input (
                        message: 'Should we deploy to prod, service will restart with downtime?'
                    )
                }
            }
            steps {
                sh "docker pull docker-staging.imio.be/renocopro/mutual:$BUILD_ID"
                sh "docker tag docker-staging.imio.be/renocopro/mutual:$BUILD_ID docker-prod.imio.be/renocopro/mutual:$BUILD_ID"
                sh "docker tag docker-staging.imio.be/renocopro/mutual:$BUILD_ID docker-prod.imio.be/renocopro/mutual:latest"
                sh "docker push docker-prod.imio.be/renocopro/mutual"
                sh "docker rmi docker-staging.imio.be/renocopro/mutual:$BUILD_ID"
                sh "docker rmi docker-prod.imio.be/renocopro/mutual:latest"
                sh "docker rmi docker-prod.imio.be/renocopro/mutual:$BUILD_ID"
                sh "mco shell run 'docker pull docker-prod.imio.be/renocopro/mutual:$BUILD_ID' -I /^site-prod10/"
                sh "mco shell run 'systemctl restart renocopro.service' --tail -I /^site-prod10/"
            }
        }
    }
}
