#!groovy


def msg
def repoFullName
def targetBranch
def prId
def prUid
def releaseId


pipeline {

    agent {
        label 'master'
    }

    triggers {
        ciBuildTrigger(
            noSquash: true,
            providerList: [
                rabbitMQSubscriber(
                    name: env.FEDORA_CI_MESSAGE_PROVIDER,
                    overrides: [
                        topic: 'org.fedoraproject.prod.pagure.pull-request.new',
                        queue: 'osci-pipelines-queue-7'
                    ],
                    checks: [
                        [field: '$.pullrequest.project.namespace', expectedValue: '^rpms$']
                    ]
                )
            ]
        )
    }

    parameters {
        string(name: 'CI_MESSAGE', defaultValue: '{}', description: 'CI Message')
    }

    stages {
        stage('Check Message') {
            steps {
                script {
                    msg = readJSON text: CI_MESSAGE

                    repoFullName = msg['pullrequest']['project']['fullname']
                    targetBranch = msg['pullrequest']['branch']
                    prId = msg['pullrequest']['id']
                    prUid = msg['pullrequest']['uid']
                    prCommit = msg['pullrequest']['commit_stop']
                    prComment = 0
                }
            }
        }

        stage('Schedule Build') {
            steps {
                // note: there is only one build pipeline for all releases
                build(
                    job: 'fedora-ci/dist-git-build-pipeline/master',
                    wait: false,
                    parameters: [
                        string(name: 'REPO_FULL_NAME', value: repoFullName),
                        string(name: 'TARGET_BRANCH', value: targetBranch),
                        string(name: 'PR_ID', value: prId),
                        string(name: 'PR_UID', value: prUid),
                        string(name: 'PR_COMMIT', value: prCommit),
                        string(name: 'PR_COMMENT', value: prComment)
                    ]
                )
            }
        }
    }
}
