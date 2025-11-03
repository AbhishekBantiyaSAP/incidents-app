@Library('piper-lib') _  // Load SAP Project Piper library

pipeline {
    agent {
        docker {
            image 'ppiper/node-browsers:latest'   // ✅ SAP-maintained image with Node, Helm, kubectl, pack, etc.
            reuseNode true
        }
    }

    environment {
        APP_NAME   = 'incidents-jenkins'
        NAMESPACE  = 'i528716-pipeline'
        REGISTRY   = 'content-agent.common.repositories.cloud.sap/i528716'
        IMAGE_TAG  = 'v1'
        CDS_CMD    = 'npx cds'
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Build Application') {
            steps {
                sh '''
                echo "=== Running CDS build ==="
                npm install @sap/cds-dk ctz
                ${CDS_CMD} build --production

                mkdir -p app/html5-deployer/resources
                npm ci --prefix app/incidents
                npm run build --prefix app/incidents

                cp -f app/incidents/dist/incidents.zip app/html5-deployer/resources/incidents.zip
                '''
            }
        }

        stage('Build & Push Containers') {
            steps {
                withCredentials([
                    dockerConfig(credentialsId: 'docker-config-json', variable: 'DOCKER_CONFIG_JSON')
                ]) {
                    script {
                        writeFile file: 'docker-config.json', text: "${DOCKER_CONFIG_JSON}"

                        echo "=== Building container images using Piper cnbBuild ==="

                        // ---- SRV ----
                        cnbBuild(
                            script: this,
                            path: 'gen/srv',
                            dockerImage: 'paketobuildpacks/builder-jammy-base',
                            buildpacks: ['paketobuildpacks/nodejs'],
                            containerRegistryUrl: "${REGISTRY}",
                            containerImageName: "${APP_NAME}-srv",
                            containerImageTag: "${IMAGE_TAG}",
                            dockerConfigJson: 'docker-config.json'
                        )

                        // ---- HANA Deployer ----
                        cnbBuild(
                            script: this,
                            path: 'gen/db',
                            dockerImage: 'paketobuildpacks/builder-jammy-base',
                            buildpacks: ['paketobuildpacks/nodejs'],
                            containerRegistryUrl: "${REGISTRY}",
                            containerImageName: "${APP_NAME}-hana-deployer",
                            containerImageTag: "${IMAGE_TAG}",
                            dockerConfigJson: 'docker-config.json'
                        )

                        // ---- HTML5 Deployer ----
                        cnbBuild(
                            script: this,
                            path: 'app/html5-deployer',
                            dockerImage: 'paketobuildpacks/builder-jammy-base',
                            buildpacks: [
                                'deploy-releases-hyperspace-docker.common.repositories.sapcloud.cn/buildpacks/application-content-deployer-buildpack:1.1.0'
                            ],
                            containerRegistryUrl: "${REGISTRY}",
                            containerImageName: "${APP_NAME}-html5-deployer",
                            containerImageTag: "${IMAGE_TAG}",
                            dockerConfigJson: 'docker-config.json'
                        )
                    }
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                withCredentials([
                    string(credentialsId: 'kube-config-secret', variable: 'KUBE_B64')
                ]) {
                    sh '''
                    mkdir -p ~/.kube
                    echo "$KUBE_B64" | base64 --decode > ~/.kube/config
                    '''

                    script {
                        helmExecute(
                            script: this,
                            helmCommand: 'upgrade',
                            chartPath: 'gen/chart',
                            namespace: "${NAMESPACE}",
                            releaseName: "${APP_NAME}",
                            additionalParameters: [
                                '--install',
                                '--set-file', 'xsuaa.jsonParameters=xs-security.json'
                            ],
                            kubeConfig: '~/.kube/config',
                            verbose: true
                        )
                    }
                }
            }
            post {
                success {
                    echo "✅ Successfully deployed ${APP_NAME} to namespace ${NAMESPACE}"
                }
                failure {
                    echo "❌ Helm deployment failed — please check the logs above."
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning workspace..."
            cleanWs()
        }
        failure {
            archiveArtifacts artifacts: '**/*.log', allowEmptyArchive: true
        }
    }
}
