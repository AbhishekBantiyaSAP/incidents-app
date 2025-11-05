// Jenkinsfile
@Library('piper-lib-os@v1.222.0') _

piperPipeline {

    general {
        buildTool        = 'npm'
        productiveBranch = 'main'
    }

    stages {
        Build {
            cnbBuild = true
        }

        // Acceptance stage is commented in the YAML → keep it disabled
        // Acceptance {
        //     deployTool            = 'helm3'
        //     deploymentName        = 'incidents-jenkins'
        //     kubernetesDeploy      = true
        //     namespace             = 'i528716-pipeline'
        //     kubeConfigFileCredentialsId = '<your-kube-config-cred-id>'
        // }
    }

    // ──────────────────────────────────────────────────────────────
    // Steps (steps: …)
    // ──────────────────────────────────────────────────────────────
    steps {
        npmExecuteScripts {
            install      = true
            runScripts   = ['production-build', 'ui-build']
        }

        cnbBuild {
            // The default image is used → no containerImageName / containerImageTag
            containerRegistryUrl = env.DOCKER_REGISTRY   // <-- set in Jenkins env
            multipleImages = [
                [
                    containerImageName: 'incidents-jenkins-srv',
                    path: 'gen/srv'
                ],
                [
                    containerImageName: 'incidents-jenkins-hana-deployer',
                    path: 'gen/db'
                ],
                [
                    containerImageName: 'incidents-jenkins-html5-deployer',
                    path: 'app/html5-deployer',
                    buildpacks: [
                        'deploy-releases-hyperspace-docker.common.repositories.sapcloud.cn/buildpacks/application-content-deployer-buildpack:1.1.0'
                    ]
                ]
            ]
            // containerImageTag is set globally for all images
            containerImageTag = 'latest'
        }
    }
}