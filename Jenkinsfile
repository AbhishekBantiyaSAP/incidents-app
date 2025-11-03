@Library('piper-lib') _  // Load SAP Project Piper library

pipeline {
    agent {
        docker {
            image 'ubuntu:24.04'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
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

        stage('Setup Tools & Logins') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'buildpack-registry-cred', usernameVariable: 'BP_USER', passwordVariable: 'BP_PASS'),
                    usernamePassword(credentialsId: 'docker-registry-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS'),
                    string(credentialsId: 'kube-config-secret', variable: 'KUBE_B64')
                ]) {
                    sh '''
                    set -euo pipefail

                    echo "=== Updating APT package lists ==="
                    apt-get update -y
                    apt-get install -y software-properties-common curl ca-certificates sudo make gnupg npm

                    echo "=== Setting up Kubernetes APT repo ==="
                    mkdir -p /etc/apt/keyrings
                    rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
                    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
                        gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
                    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
                        > /etc/apt/sources.list.d/kubernetes.list
                    apt-get update -y
                    apt-get install -y kubectl

                    echo "=== Installing Helm ==="
                    curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

                    echo "=== Installing pack CLI ==="
                    curl -sSL "https://github.com/buildpacks/pack/releases/download/v0.38.2/pack-v0.38.2-linux.tgz" | \
                        tar -C /usr/local/bin/ --no-same-owner -xzv pack

                    echo "=== Installing ctz and @sap/cds-dk globally ==="
                    npm install -g ctz @sap/cds-dk

                    echo "=== Docker login ==="
                    echo "$BP_PASS" | docker login ${BUILDPACK_REGISTRY} -u ${BP_USER} --password-stdin
                    echo "$DOCKER_PASS" | docker login ${REGISTRY} -u ${DOCKER_USER} --password-stdin

                    echo "=== Setting up Kubeconfig ==="
                    mkdir -p ~/.kube
                    echo "$KUBE_B64" | base64 --decode > ~/.kube/config

                    echo "✅ All tools and credentials configured successfully."
                    '''
                }
            }
        }

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Build & Containerize') {
            steps {
                script {
                    echo "=== Running CDS build ==="
                    sh '''
                    ${CDS_CMD} build --production

                    mkdir -p app/html5-deployer/resources
                    npm i --prefix app/incidents
                    npm run build --prefix app/incidents
                    
                    cp -f app/incidents/dist/incidents.zip app/html5-deployer/resources/incidents.zip
                    '''

                    echo "=== Building container images using Project Piper (cnbBuild) ==="
                    // ---- Build SRV ----
                    cnbBuild(
                        script: this,
                        path: 'gen/srv',
                        dockerImage: 'paketobuildpacks/builder-jammy-base',
                        buildpacks: ['paketobuildpacks/nodejs'],
                        containerRegistryUrl: "${REGISTRY}",
                        containerImageName: "${APP_NAME}-srv",
                        containerImageTag: "${IMAGE_TAG}",
                        dockerConfigJson: '/home/jenkins/.docker/config.json'
                    )

                    // ---- Build HANA DEPLOYER ----
                    cnbBuild(
                        script: this,
                        path: 'gen/db',
                        dockerImage: 'paketobuildpacks/builder-jammy-base',
                        buildpacks: ['paketobuildpacks/nodejs'],
                        containerRegistryUrl: "${REGISTRY}",
                        containerImageName: "${APP_NAME}-hana-deployer",
                        containerImageTag: "${IMAGE_TAG}",
                        dockerConfigJson: '/home/jenkins/.docker/config.json'
                    )

                    // ---- Build HTML5 DEPLOYER ----
                    cnbBuild(
                        script: this,
                        path: 'app/html5-deployer',
                        dockerImage: 'paketobuildpacks/builder-jammy-base',
                        buildpacks: ['deploy-releases-hyperspace-docker.common.repositories.sapcloud.cn/buildpacks/application-content-deployer-buildpack:1.1.0'],
                        containerRegistryUrl: "${REGISTRY}",
                        containerImageName: "${APP_NAME}-html5-deployer",
                        containerImageTag: "${IMAGE_TAG}",
                        dockerConfigJson: '/home/jenkins/.docker/config.json'
                    )

                    echo "✅ Build & containerization complete."
                }
            }
        }

        // stage('Deploy with Helm') {
        //     steps {
        //         sh '''
        //         echo "=== Deploying to Kubernetes via Helm ==="
        //         helm upgrade --install ${APP_NAME} \
        //             --namespace ${NAMESPACE} \
        //             ./gen/chart \
        //             --set-file xsuaa.jsonParameters=xs-security.json \
        //             --wait --timeout 5m
        //         '''
        //     }
        //     post {
        //         success {
        //             echo "✅ Successfully deployed ${APP_NAME} to namespace ${NAMESPACE}"
        //         }
        //         failure {
        //             echo "❌ Helm deployment failed — please check the logs above."
        //         }
        //     }
        // }

        stage('Deploy with Helm (Piper helmExecute)') {
            steps {
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
            post {
                success {
                    echo "✅ Successfully deployed ${APP_NAME} to namespace ${NAMESPACE}"
                }
                failure {
                    echo "❌ Deployment failed — check Helm logs"
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
