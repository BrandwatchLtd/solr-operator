buildTag = "build.${env.BUILD_NUMBER}"
gcpProject = "bw-prod-platform0"
imageName = "eu.gcr.io/${gcpProject}/solr-operator"

def buildAndDeployDockerImage() {
    hadolint("build/Dockerfile.build")

    gitSha = sh (script: "git rev-parse HEAD", returnStdout: true)
    version = sh (script: "date '+%Y%m%d.%H%M%S'", returnStdout: true)

    docker.withRegistry("https://eu.gcr.io", "gcr:${gcpProject}") {
        sh """ docker build \
            --build-arg REPO=${gcpProject} \
            --build-arg VERSION=${version} \
            --build-arg GIT_SHA=${gitSha} \
            -t ${gcpProject}/solr-operator:${branchName} .
        """
        img.push()
        img.push(buildTag)
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

    if (env.BRANCH_NAME == "main") {

      stage("Deploy docker image") {
          buildAndDeployDockerImage()
      }

      stage("Deploy helm chart") {
          def chartVersion = deployHelmChart("helm/solr-operator", "bw-prod-platform0")
          slipstreamDeploy("solr-operator", chartVersion)
      }
    }
}
