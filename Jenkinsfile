#!groovy

import groovy.json.JsonSlurperClassic

def msg
def artifactId
def allTaskIds = [] as Set
def allBuilds = [:]

def productVersion = 'fedora-35'

pipeline {

    agent none

    options {
        buildDiscarder(logRotator(daysToKeepStr: '45', artifactNumToKeepStr: '100'))
        skipDefaultCheckout()
    }

    // triggers {
    //    ciBuildTrigger(
    //        noSquash: true,
    //        providerList: [
    //            rabbitMQSubscriber(
    //                name: env.FEDORA_CI_MESSAGE_PROVIDER,
    //                overrides: [
    //                    topic: 'org.fedoraproject.prod.bodhi.update.status.testing.koji-build-group.build.complete',
    //                    queue: 'osci-pipelines-queue-0'
    //                ],
    //                checks: [
    //                    [field: '$.artifact.release', expectedValue: '^f35$']
    //                ]
    //            )
    //        ]
    //    )
    // }

    parameters {
        string(name: 'CI_MESSAGE', defaultValue: '{}', description: 'CI Message')
    }

    stages {
        stage('Trigger Testing') {
            steps {
                script {
                    msg = readJSON text: params.CI_MESSAGE

                    if (msg) {
                        msg['artifact']['builds'].each { build ->
                            allTaskIds.add(build['task_id'])
                            allBuilds[build['task_id']] = build['nvr']
                        }
                        def releaseId = msg['artifact']['release']

                        def nvr
                        def gatingDecisionResponse
                        def gatingDecision
                        def retryCounter
                        def reqs
                        def packages

                        if (allTaskIds) {
                            allTaskIds.each { taskId ->
                                artifactId = "koji-build:${taskId}"
                                nvr = allBuilds[taskId]
                                packages = [] as Set

                                retryCounter = 0
                                retry(10) {
                                    // retry Greenwave query up to 10 times
                                    if (retryCounter) {
                                        // sleep 10 seconds if this is not a first attempt
                                        sleep(time: 10, unit: 'SECONDS')
                                    }
                                    retryCounter += 1

                                    echo "Querying greenwave for ${nvr} ..."
                                    gatingDecisionResponse = httpRequest(
                                        url: env.FEDORA_CI_GREENWAVE_API_URL + '/decision',
                                        httpMode: 'POST',
                                        acceptType: 'APPLICATION_JSON',
                                        contentType: 'APPLICATION_JSON',
                                        validResponseCodes: '200',
                                        consoleLogResponseBody: false,
                                        requestBody: """
                                            {
                                                "decision_context": "bodhi_update_push_stable",
                                                "product_version": "fedora-35",
                                                "subject_type": "koji_build",
                                                "subject_identifier": "${nvr}",
                                                "verbose": false
                                            }
                                        """
                                    )
                                }
                                gatingDecision = new JsonSlurperClassic().parseText(gatingDecisionResponse.content)

                                reqs = gatingDecision.get('satisfied_requirements') + gatingDecision.get('unsatisfied_requirements')
                                reqs.each { req ->
                                    if (req.get('testcase') == 'fedora-ci.koji-build.scratch-build.validation') {
                                        if (req.get('scenario')) {
                                            packages.add(req.get('scenario'))
                                        }
                                    }
                                }

                                if (packages) {
                                    packages.each { pkg ->
                                        build(
                                            job: 'fedora-ci/dist-git-build-pipeline/scratch-rebuild-wip',
                                            wait: false,
                                            parameters: [
                                                string(name: 'ARTIFACT_ID', value: artifactId),
                                                string(name: 'PACKAGE_NAME', value: pkg),
                                                string(name: 'TEST_PROFILE', value: releaseId)
                                            ]
                                        )
                                    }
                                } else {
                                    echo "The rebuild test is not enabled for any of the components in this update..."
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
