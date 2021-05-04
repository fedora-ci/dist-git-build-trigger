#!groovy

retry (10) {
    // load pipeline configuration into the environment
    httpRequest("${FEDORA_CI_PIPELINES_CONFIG_URL}/environment").content.split('\n').each { l ->
        l = l.trim(); if (l && !l.startsWith('#')) { env["${l.split('=')[0].trim()}"] = "${l.split('=')[1].trim()}" }
    }
}


def config = [
    "f33": [
        "product_version": "fedora-33"
    ],
    "f34": [
        "product_version": "fedora-34"
    ],
    "f35": [
        "product_version": "fedora-35"
    ],
    "f36": [
        "product_version": "fedora-36"
    ],
    "f37": [
        "product_version": "fedora-37"
    ],
    "f38": [
        "product_version": "fedora-38"
    ],
    "f39": [
        "product_version": "fedora-39"
    ]
]


def msg
def artifactId
def allTaskIds = [] as Set
def allBuilds = [:]


pipeline {

    agent none

    libraries {
        lib("fedora-pipeline-library@08813c663ff9a05335ed50ba17a0a66cbcd4f971")
    }

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
    //                    [field: '$.artifact.release', expectedValue: '^f[3-9]{1}[2-9]{1}$']
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
                    def packages = [] as Set

                    if (msg) {
                        msg['artifact']['builds'].each { build ->
                            allTaskIds.add(build['task_id'])
                            allBuilds[build['task_id']] = build['nvr']
                        }
                        def releaseId = msg['artifact']['release']
                        def productVersion = config[releaseId]['product_version']
                        def nvr
                        def filteredReqs

                        if (allTaskIds) {
                            allTaskIds.each { taskId ->
                                artifactId = "koji-build:${taskId}"
                                nvr = allBuilds[taskId]

                                echo "Querying greenwave for ${nvr} ..."
                                filteredReqs = getGatingRequirements(
                                    artifactId: artifactId,
                                    decisionContext: 'bodhi_update_push_testing',
                                    productVersion: productVersion,
                                    scenarioPrefix: 'rebuild/',
                                    testcase: 'fedora-ci.koji-build.scratch-build.validation'
                                )
                                echo "filtered: ${filteredReqs}"

                                filteredReqs.each { req ->
                                    packages.add(req.get('scenario').split('/')[1])
                                }
                            }
                        }

                        if (packages) {
                            packages.each { pkg ->
                                build(
                                    job: 'fedora-ci/dist-git-build-pipeline/scratch-rebuild',
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
