k8sVersion = "master"
buildTag = "build.${env.BUILD_NUMBER}"
gcpProject = "bw-prod-platform0"

node {
    stage('Checkout') {
        checkout([
            $class: 'GitSCM',
            branches: scm.branches,
            doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
            extensions: scm.extensions + [[$class: 'CleanCheckout']],
            userRemoteConfigs: scm.userRemoteConfigs
        ])

        versions = readJSON file: "versions.json"
        version = versions["version"]
        buildNumber = versions["build_number"]
    }

    stage("Validate chart") {
        kubeval.validateChart("helm/solr-operator")
    }

    if (env.BRANCH_NAME == "master") {
      stage("Deploy helm chart") {
          def chartVersion = deployHelmChart("helm/solr-operator", "bw-prod-platform0")
      }
    }
}
