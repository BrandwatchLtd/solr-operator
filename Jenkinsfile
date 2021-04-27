branchName = env.BRANCH_NAME.replace('/', '_')
buildTag = (env.BRANCH_NAME == "main") ? "build.${env.BUILD_NUMBER}" : "build.${env.BUILD_NUMBER}.branchName"
gcpProject = "bw-prod-platform0"
imageName = "eu.gcr.io/${gcpProject}/solr-operator"

def buildAndDeployDockerImage() {
    hadolint("build/Dockerfile")

    gitSha = sh (script: "git rev-parse HEAD", returnStdout: true)

    docker.withRegistry("https://eu.gcr.io", "gcr:${gcpProject}") {

        sh """ docker build \
            --build-arg GIT_SHA="${gitSha}" \
            -t ${imageName}:${buildTag} \
            -f ./build/Dockerfile .
        """

        if (env.BRANCH_NAME == "main") {
            img = docker.image("${imageName}:${buildTag}")
            img.push()
            img.push("latest")
        }
    }
}

def slipstreamDeploy(app, chartVersion = null) {
    echo "Performing slipstream deploy of ${app} and helm chart ${chartVersion}"

    def payload = []
    if (chartVersion) {
        payload = [ helmChart: [ version: chartVersion ] ]
    }

    echo "Slipstream payload ${payload}"
    // Do slipstream stuff
    pushSlipstreamMetadata(
            gcpProject,
            "platform",
            app,
            buildTag,
            payload,
            "${imageName}:${buildTag}"
    )

    withGCloudCredentials(gcpProject) {
        def ssHome = tool 'slipstream'
        withEnv(["PATH+SS=${ssHome}"]) {
            sh (
                label: "Notify slipstream and deploy to stage",
                script: "slipstream deploy stage platform ${app} --tag ${buildTag} --wait --quiet"
            )
        }
    }
}

node {
    stage('Checkout') {
        checkout([
            $class: 'GitSCM',
            branches: scm.branches,
            doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
            extensions: scm.extensions + [[$class: 'CleanCheckout']],
            userRemoteConfigs: scm.userRemoteConfigs
        ])
    }

    stage("Validate chart") {
        kubeval.validateChart("helm/solr-operator")
    }

    docker_stage = (env.BRANCH_NAME == "main") ? "Deploy docker image" : "Build docker image"
    stage(docker_stage) {
        buildAndDeployDockerImage()
    }

    if (env.BRANCH_NAME == "main") {
      stage("Deploy helm chart") {
          def chartVersion = deployHelmChart("helm/solr-operator", "bw-prod-platform0")
          slipstreamDeploy("solr-operator", chartVersion)
      }
    }
}
