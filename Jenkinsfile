@Library('piper-lib-os') _

pipeline {
    agent any

    stages {
        stage('Run Piper') {
            steps {
                script {
                    checkout scm
                    
                    piperPipeline script: this
                }
            }
        }
    }
}
