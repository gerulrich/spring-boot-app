pipeline {
    agent any
    tools {
        jdk 'java15amz'
        maven 'mvn3.5.3'
    }

    environment {
        RELEASE_BRANCH = "master"
        CREDENTIAL_ID = "1d1e8146-73e0-4936-abc4-e6b92e7a18c7"
        CLUSTER_BETA = "ppt-beta"
    }

    stages {
        stage ('Initialize') {
            steps {
                sh '''
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
                    echo "JENKINS_HOME = ${JENKINS_HOME}"
                    echo "JAVA_HOME = ${JAVA_HOME}"
                    '''
                script {
                    def jenkinsProperties = [buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '2'))]
                    if (env.BRANCH_NAME == 'develop') {
                        echo 'Using development configuration'
                        jenkinsProperties.add(pipelineTriggers([[$class: "SCMTrigger", scmpoll_spec: "H/5 * * * *"], ]))
                    }
                    properties(jenkinsProperties)
                }
            }
        }

        stage('Ask Version') {
            when {
                branch RELEASE_BRANCH
            }

            steps {
                script {
                    //Get current version
                    originalVersion = readMavenPom().getVersion()
                    if (!originalVersion.endsWith("-SNAPSHOT")) {
                        error("Trying to release a non-snapshot version")
                    }

                    // Calculate versions
                    releaseVersion = originalVersion.substring(0, originalVersion.size() - "-SNAPSHOT".size())
                    versionConstituents = releaseVersion.split("\\.")
                    versionConstituents[-1] = "${versionConstituents[-1].toInteger() + 1}".toString()
                    newSnapshotVersion = versionConstituents.join('.') + "-SNAPSHOT"
                }


                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        userInput = input(
                            id: 'userInput', message: 'VERSION', parameters: [
                            [$class: 'TextParameterDefinition', defaultValue: releaseVersion, description: 'Version to release', name: 'RELEASE_VERSION'],
                            [$class: 'TextParameterDefinition', defaultValue: newSnapshotVersion, description: 'Version to keep on repository to continue development', name: 'NEW_SNAPSHOT_VERSION']
                        ])
                    }
                }


                script {
                    releaseVersion = userInput['RELEASE_VERSION']
                    newSnapshotVersion = userInput['NEW_SNAPSHOT_VERSION']
                     //Change current build name
                     currentBuild.description = "Release $releaseVersion"
                     echo "Release Version: $releaseVersion"
                     echo "New snapshot version: $newSnapshotVersion"
                }
            }
        }

        stage('Test') {
            steps {
                sh "mvn --batch-mode -V -U -e clean test -Dsurefire.useFile=false"
            }
        }

        stage('Sonar scan execution') {
            // Run the sonar scan
            steps {
                script {
                    withSonarQubeEnv {
                        sh "mvn'  verify sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"
                    }
                }
            }
        }

        // waiting for sonar results based into the configured web hook in Sonar server which push the status back to jenkins
        stage('Sonar scan result check') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    retry(3) {
                        script {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
            }
        }

        stage('Bump release versions') {
            when {
                branch RELEASE_BRANCH
            }
            steps {
                sshagent(credentials:[CREDENTIAL_ID]) {
                    sh "mvn versions:set -DgenerateBackupPoms=false -DnewVersion=$releaseVersion"
                    sh "mvn -e clean compile scm:checkin scm:tag -Dmessage=\"SCM - version $releaseVersion\" -Dtag=$releaseVersion"
                    sh "mvn versions:set scm:checkin -DgenerateBackupPoms=false -DnewVersion=$newSnapshotVersion -Dmessage=\"SCM - new dev version $newSnapshotVersion\""
                }
            }
        }

        stage('Empacar') {
            when {
                branch RELEASE_BRANCH
            }
            steps {
                sh "$JENKINS_HOME/empacar.py -u $EMPACAR_USER -p $EMPACAR_PASS -r lenny -t $releaseVersion"
            }
        }

        stage('Deploy to BETA') {
            when {
                branch RELEASE_BRANCH
            }
            steps {
                script {
                    ARTIFACT_VERSION = "com.despegar.loyalty:ppt:$releaseVersion"
                    echo "Deploying artifact version: $ARTIFACT_VERSION"
                    sh "$JENKINS_HOME/deploy.py -l prod -n nexusprod -a $ARTIFACT_VERSION -c $CLUSTER_BETA -u $CLOUDIA_CLIENT_ID -p $CLOUDIA_CLIENT_SECRET"
                }
            }
        }
    }
}

