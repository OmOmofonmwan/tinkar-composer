#!groovy
@Library("titan-library") _

pipeline {
    agent any
    tools {
       jdk "java-21"
       maven 'default'
       git 'git'
    }

    environment {
        GPG_PASSPHRASE      = credentials('gpg_passphrase')
        BRANCH_NAME         = "${GIT_BRANCH.split("/").size() > 1 ? GIT_BRANCH.split("/")[1] : GIT_BRANCH}"
    }

    triggers {
        gitlab(triggerOnPush: true, triggerOnMergeRequest: true, branchFilterType: 'All')
    }

    options {
        // Set this to true if you want to clean workspace during the prep stage
        skipDefaultCheckout(false)
        // Console debug options
        timestamps()
        ansiColor('xterm')
        // necessary for communicating status to gitlab
        gitLabConnection('fda-shield-group')
    }

    stages {
        stage('Initialize') {
            steps{
                //Clean before checkout / build
                cleanWs()
                checkout scm
                sh """
                    java --version
                    mvn --version
                    printenv
                """
            }
        }

        stage('Maven Build') {
            when {
                expression { return BRANCH_NAME == "main"}
            }
            steps {
                updateGitlabCommitStatus name: 'build', state: 'running'
                script{
                    configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) {
                        sh """
                            mvn clean install -s '${MAVEN_SETTINGS}' \
                                --batch-mode \
                                -e \
                                -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn \
                                -Dmaven.build.cache.enabled=false \
                                -PcodeQuality
                        """
                    }
                }
            }
        }

        stage('Maven Build -- Feature Branch') {
            when {
                expression { return BRANCH_NAME != "main"}
            }
            steps {
                updateGitlabCommitStatus name: 'build', state: 'running'
                script{
                    configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) {
                        sh """
                            mvn clean install -s '${MAVEN_SETTINGS}' \
                                --batch-mode \
                                -e \
                                -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn \
                                -Dmaven.build.cache.enabled=false \
                                -Denforcer.skip=true \
                                -PcodeQuality
                        """
                    }
                }
            }
        }

        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    pomModel = readMavenPom(file: 'pom.xml')
                    pomVersion = pomModel.getVersion()
                    isSnapshot = pomVersion.contains("-SNAPSHOT")
                    repositoryId = 'maven-releases'

                    if (isSnapshot) {
                        repositoryId = 'maven-snapshots'
                    }

                    configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) {
                        sh """
                            mvn deploy \
                            --batch-mode \
                            -e \
                            -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn \
                            -DskipTests \
                            -DskipITs \
                            -Dmaven.main.skip \
                            -Dmaven.test.skip \
                            -s '${MAVEN_SETTINGS}' \
                            -P inject-application-properties \
                            -Dmaven.build.cache.enabled=false \
                            -DrepositoryId='${repositoryId}' \
                            -Dgpg.passphrase='${GPG_PASSPHRASE}'
                        """
                    }
                }
            }
        }

        stage("Publish to OSSRH maven central") {
            when{
                expression{ return buildingTag() && !isSnapshot }
            }
            steps {
                script {
                    pomModel = readMavenPom(file: 'pom.xml')
                    pomVersion = pomModel.getVersion()
                    isSnapshot = pomVersion.contains("-SNAPSHOT")
                    repositoryId = 'maven-snapshots'

                    if (env.TAG_NAME) {
                        if (!isSnapshot) {
                            repositoryId = 'maven-releases'
                        } else {
                            echo "ERROR: Only tag release versions. Tagged version was '${pomVersion}'"
                            fail()
                        }
                    }

                    configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) {
                        sh """
                            mvn deploy \
                                --batch-mode \
                                -e \
                                -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn \
                                -Dmaven.build.cache.enabled=false \
                                -DskipTests \
                                -DskipITs \
                                -Dmaven.main.skip \
                                -Dmaven.test.skip \
                                -s '${MAVEN_SETTINGS}' \
                                -DrepositoryId='${repositoryId}' \
                                -DrepositoryIdOSSRH='true' \
                                -PsourceJavadocOSSRH,stageOSSRH -Dgpg.passphrase='${GPG_PASSPHRASE}'
                        """
                    }
                }
            }
        }
    }

    post {
        failure {
            updateGitlabCommitStatus name: 'build', state: 'failed'
            emailext(
                recipientProviders: [requestor(), culprits()],
                subject: "Build failed in Jenkins: ${env.JOB_NAME} - #${env.BUILD_NUMBER}",
                body: """
                    Build failed in Jenkins: ${env.JOB_NAME} - #${BUILD_NUMBER}

                    See attached log or URL:
                    ${env.BUILD_URL}
                """,
                attachLog: true
            )
        }
        aborted {
            updateGitlabCommitStatus name: 'build', state: 'canceled'
        }
        unstable {
            updateGitlabCommitStatus name: 'build', state: 'failed'
            emailext(
                subject: "Unstable build in Jenkins: ${env.JOB_NAME} - #${env.BUILD_NUMBER}",
                body: """
                    See details at URL:
                    ${env.BUILD_URL}
                """,
                attachLog: true
            )
        }
        changed {
            updateGitlabCommitStatus name: 'build', state: 'success'
            emailext(
                recipientProviders: [requestor(), culprits()],
                subject: "Jenkins build is back to normal: ${env.JOB_NAME} - #${env.BUILD_NUMBER}",
                body: """
                    Jenkins build is back to normal: ${env.JOB_NAME} - #${env.BUILD_NUMBER}

                    See URL for more information:
                    ${env.BUILD_URL}
                """
            )
        }
        success {
            updateGitlabCommitStatus name: 'build', state: 'success'
        }
        cleanup {
            // Clean the workspace after build
            cleanWs(cleanWhenNotBuilt: false,
                deleteDirs: true,
                disableDeferredWipeout: true,
                notFailBuild: true,
                patterns: [
                [pattern: '.gitignore', type: 'INCLUDE']
            ])
        }
    }
}
