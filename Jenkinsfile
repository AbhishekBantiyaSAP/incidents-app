@Library('piper-lib-os') _

pipeline {
    agent any

    stages {
        stage('Run Piper') {
            steps {
                script {
                    piperPipeline script: this
                }
            }
        }
    }
}
