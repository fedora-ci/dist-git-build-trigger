#!groovy

def msg
def artifactId

def triggerComponents = ['glibc'] as Set
def rebuildComponents = ['kernel'] as Set


pipeline {

    agent {
        label 'dist-git-build-trigger'
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
    //                    queue: 'osci-pipelines-queue-10'
    //                ],
    //                checks: [
    //                    [field: '$.artifact.release', expectedValue: '^f34$']
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
                        def releaseId = msg['artifact']['release']

                        msg['artifact']['builds'].any { kojiBuild ->
                            if (kojiBuild['component'] in triggerComponents) {
                                artifactId = "koji-build:${kojiBuild['task_id']}"

                                rebuildComponents.each { component ->
                                    build(
                                        job: 'fedora-ci/dist-git-build-pipeline/master',
                                        wait: false,
                                        parameters: [
                                            string(name: 'ARTIFACT_ID', value: artifactId),
                                            string(name: 'BUILD_TARGET', value: "${releaseId}-updates-testing-pending"),
                                            string(name: 'TEST_SCENARIO', value: component),
                                            string(name: 'REPO_FULL_NAME', value: "rpms/${component}")
                                        ]
                                    )
                                }
                                return true  // break
                            }
                        }
                    }
                }
            }
        }
    }
}
