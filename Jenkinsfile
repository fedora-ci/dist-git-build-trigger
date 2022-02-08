#!groovy

retry (10) {
    // load pipeline configuration into the environment
    httpRequest("${FEDORA_CI_PIPELINES_CONFIG_URL}/environment").content.split('\n').each { l ->
        l = l.trim(); if (l && !l.startsWith('#')) { env["${l.split('=')[0].trim()}"] = "${l.split('=')[1].trim()}" }
    }
}

def msg
def repoFullName
def sourceRepoFullName
def targetBranch
def prId
def prUid
def releaseId
def namespace


pipeline {

    agent {
        label 'dist-git-build-trigger'
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '45', artifactNumToKeepStr: '100'))
        skipDefaultCheckout()
    }

    triggers {
        ciBuildTrigger(
            noSquash: true,
            providerList: [
                rabbitMQSubscriber(
                    name: env.FEDORA_CI_MESSAGE_PROVIDER,
                    overrides: [
                        topic: 'org.fedoraproject.prod.pagure.pull-request.comment.added',
                        queue: 'osci-pipelines-queue-4'
                    ],
                    checks: [
                        [field: '$.pullrequest.project.namespace', expectedValue: 'rpms|tests']
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

                    if (!msg) {
                        currentBuild.result = 'ABORTED'
                        error('No messages, nothing to do.')
                    }

                    def lastComment = msg['pullrequest']['comments'][-1]
                    def lastCommentText = lastComment['comment'].trim()

                    if (msg['pullrequest']['status'] != 'Open') {
                        currentBuild.result = 'ABORTED'
                        error('The pull-request is already closed, skipping...')
                    }

                    if (!lastCommentText.contains('[citest]') && !lastComment['notification']) {
                        currentBuild.result = 'ABORTED'
                        error("The comment is not a notification or it doesn't match '[citext]'")
                    }

                    repoFullName = msg['pullrequest']['project']['fullname']
                    sourceRepoFullName = msg['pullrequest']['repo_from']['fullname']
                    targetBranch = msg['pullrequest']['branch']
                    prId = msg['pullrequest']['id']
                    prUid = msg['pullrequest']['uid']
                    prCommit = msg['pullrequest']['commit_stop']
                    prComment = lastComment['id']

                    namespace = msg['pullrequest']['project']['namespace']
                }
            }
        }

        stage('Schedule Build') {
            steps {
                script {
                    if (namespace == 'tests') {
                        // This pull-request only modifies tests - no code changes,
                        // so we don't need to scratch-build anything
                        build(
                            job: 'fedora-ci/dist-git-pipeline/master',
                            wait: false,
                            parameters: [
                                string(name: 'ARTIFACT_ID', value: "()->fedora-dist-git:${prUid}@${prCommit}#${prComment}"),
                                string(name: 'TEST_REPO_URL', value: "${env.FEDORA_CI_PAGURE_DIST_GIT_URL}/${sourceRepoFullName}#${prCommit}"),
                                string(name: 'TEST_PROFILE', value: "${env.FEDORA_CI_RAWHIDE_RELEASE_ID}")
                            ]
                        )
                    } else {
                        build(
                            job: 'fedora-ci/dist-git-build-pipeline/master',
                            wait: false,
                            parameters: [
                                string(name: 'REPO_FULL_NAME', value: "${repoFullName}"),
                                string(name: 'SOURCE_REPO_FULL_NAME', value: "${sourceRepoFullName}"),
                                string(name: 'TARGET_BRANCH', value: "${targetBranch}"),
                                string(name: 'PR_ID', value: "${prId}"),
                                string(name: 'PR_UID', value: "${prUid}"),
                                string(name: 'PR_COMMIT', value: "${prCommit}"),
                                string(name: 'PR_COMMENT', value: "${prComment}")
                            ]
                        )
                    }
                }
            }
        }
    }
}
