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

                stage('TRIAL: Buildah cnbBuild (srv only)') {
            when {
                expression { true } // ← SET TO `true` TO RUN
            }
            steps {
                withCredentials([file(credentialsId: 'docker-config-json', variable: 'DOCKER_CONFIG')]) {
                    sh '''
                    set +x  # Hide sensitive output
                    echo "=== Installing Buildah in agent ==="
                    if ! command -v buildah >/dev/null; then
                        echo "Installing buildah..."
                        sudo apt-get update
                        sudo apt-get install -y buildah fuse-overlayfs slirp4netns
                        sudo rm -rf /var/lib/apt/lists/*
                    else
                        echo "buildah already installed"
                    fi

                    # Setup auth for buildah
                    mkdir -p ~/.docker
                    cp "$DOCKER_CONFIG" ~/.docker/config.json
                    echo "config.json copied for buildah"

                    # Verify buildah can see auth
                    echo "Testing buildah login..."
                    buildah login --get-login ${REGISTRY} && echo "Logged in to ${REGISTRY}" || echo "No prior login"

                    echo "=== Running cnbBuild with Buildah (srv only) ==="
                    '''
                    
                    script {
                        cnbBuild(
                            script: this,
                            path: 'gen/srv',
                            buildpacks: ['paketobuildpacks/nodejs:latest'],
                            containerRegistryUrl: "${REGISTRY}",
                            containerImageName: "${APP_NAME}-srv-buildah",
                            containerImageTag: "${IMAGE_TAG}",
                            dockerConfigJsonCredentialsId: 'docker-config-json',
                            buildEnvVars: [
                                PACK_BUILDER_DRIVER: 'buildah'
                            ]
                        )
                    }
                }
            }
            post {
                success {
                    echo "Buildah cnbBuild succeeded! Image: ${REGISTRY}/${APP_NAME}-srv-buildah:${IMAGE_TAG}"
                }
                failure {
                    echo "Buildah failed — check logs above. Docker path still works."
                }
            }
        }

        stage('Build & Push Containers') {
            steps {
                    script {
                        
                        echo "=== Building container images using Piper cnbBuild ==="

                        // ---- SRV ----
                        cnbBuild(
                            script: this,
                            path: 'gen/srv',
                            buildpacks: ['paketobuildpacks/nodejs'],
                            containerRegistryUrl: "${REGISTRY}",
                            containerImageName: "${APP_NAME}-srv",
                            containerImageTag: "${IMAGE_TAG}",
                            dockerConfigJsonCredentialsId: 'docker-config-json'
                        )

                        // ---- HANA Deployer ----
                        cnbBuild(
                            script: this,
                            path: 'gen/db',
                            buildpacks: ['paketobuildpacks/nodejs'],
                            containerRegistryUrl: "${REGISTRY}",
                            containerImageName: "${APP_NAME}-hana-deployer",
                            containerImageTag: "${IMAGE_TAG}",
                            dockerConfigJsonCredentialsId: 'docker-config-json'
                        )

                        // ---- HTML5 Deployer ----
                        cnbBuild(
                            script: this,
                            path: 'app/html5-deployer',
                            buildpacks: [
                                'deploy-releases-hyperspace-docker.common.repositories.sapcloud.cn/buildpacks/application-content-deployer-buildpack:1.1.0'
                            ],
                            containerRegistryUrl: "${REGISTRY}",
                            containerImageName: "${APP_NAME}-html5-deployer",
                            containerImageTag: "${IMAGE_TAG}",
                            dockerConfigJsonCredentialsId: 'docker-config-json'
                        )
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
