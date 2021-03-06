import groovy.json.JsonSlurperClassic

nl('docker', [time: 60, time_unit: 'MINUTES', finally: {
    slack.simpleAlert("*Job* #<${env.BUILD_URL}console|`${env.BUILD_NUMBER}`>: build ${env.JOB_NAME.replace('%2F', '/')}")
}]) {

    def paths = [:]
    def images = [:]
    def gitVars, version

    step('Initialize') {
        properties(
                [
                        buildDiscarder(logRotator(artifactDaysToKeepStr: '10', artifactNumToKeepStr: '30', daysToKeepStr: '10', numToKeepStr: '30'))
                ]
        )

        dl.login(env.ID_LOGIN_PASS_REGISTRY)
    }

    step('Checkout') {

        gitVars = checkout([
                $class                           : 'GitSCM',
                branches                         : scm.branches,
                userRemoteConfigs                : scm.userRemoteConfigs,
                doGenerateSubmoduleConfigurations: scm.doGenerateSubmoduleConfigurations,
                extensions                       : [[$class: 'CloneOption', noTags: false, shallow: false, depth: 0, reference: '']],
        ])
    }

    step('Build') {
        paths = sh(returnStdout: true, script: "find . -name Dockerfile -type f -exec dirname {} \\;").split()

        for (int i = 0; i < paths.size(); i++) {

            version = sh(returnStdout: true, script: "cat ${paths[i]}/Dockerfile | grep -Eow \"^ARG VERSION='.*'\" | grep -Po \"(?<=')[^']+(?=')\"").trim()

            images.putAt(i, docker.build("${env.ID_LOGIN_PASS_REGISTRY}/${env.REGISTRY_NAMESPACE}/nginx-php:${paths[i].substring(2).replace('/', '-')}-${version}", " \
                --label org.label-schema.schema-version=1.0 \
                --label org.label-schema.vendor='Norse Digital' \
                --label org.label-schema.name='Core Images' \
                --label org.label-schema.description='-' \
                --label org.label-schema.url='-' \
                --label org.label-schema.version=${version} \
                --label org.label-schema.vcs-ref=`git rev-parse HEAD` \
                --label org.label-schema.vcs-url=`git config remote.origin.url` \
                --label org.label-schema.build-date=`date -u +'%Y-%m-%dT%H:%M:%SZ'` \
                --build-arg COMMON_ROOTFS_DIR=./common \
                --build-arg ROOTFS_DIR=${paths[i]}/rootfs \
                --pull -f ${paths[i]}/Dockerfile . \
            "))
        }
    }

    step('Push', [retries: 2, last: true]) {
        for (int i = 0; i < images.size(); i++) {
            images[i].push()
        }
    }
}