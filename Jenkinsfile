@Library('piper-lib_os') _

pipeline {
    agent any

    environment {
    
        DOCKER_REGISTRY = credentials('DOCKER_REGISTRY')
    }

    stages {
        stage('Piper Pipeline') {
            steps {
                script {
                    // This triggers Piper and passes this script context
                    piperPipeline script: this
                }
            }
        }
    }
}
