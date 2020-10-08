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
                        topic: 'org.fedoraproject.prod.pagure.pull-request.comment.added',
                        queue: 'osci-pipelines-queue-8'
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

                    def lastComment = msg['pullrequest']['comments'][-1]
                    def lastCommentText = lastComment['comment'].trim()

                    if (msg['pullrequest']['closed_at'] != null) {
                        currentBuild.result = 'ABORTED'
                        error('The pull-request is already closed, skipping...')
                    }

                    if (lastCommentText != '[citest]' && !lastComment['notification']) {
                        currentBuild.result = 'ABORTED'
                        error("The comment is not a notification or it doesn't match '[citext]'")
                    }

                    repoFullName = msg['pullrequest']['project']['fullname']
                    targetBranch = msg['pullrequest']['branch']
                    prId = msg['pullrequest']['id']
                    prUid = msg['pullrequest']['uid']
                    prCommit = msg['pullrequest']['commit_stop']
                    prComment = lastComment['id']
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