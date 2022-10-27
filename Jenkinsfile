@Library('jenkins-common-library')

//Instanciate Objects from Libs
def util = new libs.utils.Util()

// initialize variables in case the token is used
def String OCP_CRED_PSW = ''
def String OCP_CRED_USR = ''

// Version from Maistra - need to be improved!
def MAISTRA_VERSION = "2.3"

// Parameters to be used on job
properties([
    buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '20')),
    parameters([
        string(
            name: 'OCP_API_URL',
            defaultValue: '',
            description: 'OCP Server URL'
        ),
        string(
            name: 'OCP_CRED_ID',
            defaultValue: 'jenkins-ocp-auto',
            description: 'Jenkins credentials ID for OCP cluster. When set this, it takes precedence over OCP_TOKEN.'
        ),
        string(
            name: 'OCP_TOKEN',
            defaultValue: '',
            description: 'OCP token. When OCP_CRED_ID parameter is not empty, OCP_TOKEN will be ignored.'
        ),
        choice(
            name: 'OCP_SAMPLE_ARCH',
            choices: ['x86','p', 'z', 'arm'],
            description: 'This is a switch for bookinfo images on different platforms'
        ),
        string(
            name: 'TEST_CASE',
            defaultValue: '',
            description: 'test case name, e.g. T1, T2. See tests/test_cases.go, default empty value will run all test cases.'
        ),
        choice(
            name: 'ROSA',
            choices: ['false', 'true'],
            description: 'Testing on ROSA'
        ),
        string(
            name: 'NIGHTLY',
            defaultValue: 'false',
            description: 'Install nightly operators from quay.io/maistra'
        )
    ])
])

if (OCP_API_URL == "") {
      // Define the build name and informations about it
      currentBuild.displayName = "Not Applicable"
      currentBuild.description = "Need more info"

      echo "Need to inform obrigatory fields!"

} else {

    node('jaeger'){
        // Define the build name and informations about it
        currentBuild.description = util.htmlDescription(util.whoBuild(util.getWhoBuild()))

        // Workspace cleanup and git checkout
        gitSteps()
        stage("Login in Openshift"){
            if (OCP_CRED_ID != ""){
                echo "Using ${OCP_CRED_ID} credentials ID instead of token parameter."

                withCredentials([usernamePassword(credentialsId: OCP_CRED_ID, passwordVariable: 'Password', usernameVariable: 'Username')]) {
                    OCP_CRED_PSW = Password
                    OCP_CRED_USR = Username
                    sh "oc login ${params.OCP_API_URL} -u=${OCP_CRED_USR} -p=${OCP_CRED_PSW} --insecure-skip-tls-verify"
                }
            } else {
                sh "oc login ${params.OCP_API_URL} --token=${params.OCP_TOKEN} --insecure-skip-tls-verify"
            }
        }
        stage("Start running all tests"){
            dir('tests') {
            wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[var: 'OCP_CRED_PSW', password: OCP_CRED_PSW]], varMaskRegexes: []]) {
                def OUT = sh (
                script: """
                    if [ -z "${params.TEST_CASE}" ]; 
                    then docker run \
                    --name maistra-test-tool-${env.BUILD_NUMBER} \
                    -d \
                    --rm \
                    --pull always \
                    -e SAMPLEARCH='${params.OCP_SAMPLE_ARCH}' \
                    -e OCP_CRED_USR='${OCP_CRED_USR}' \
                    -e OCP_CRED_PSW='${OCP_CRED_PSW}' \
                    -e OCP_TOKEN='${params.OCP_TOKEN}' \
                    -e OCP_API_URL='${params.OCP_API_URL}' \
                    -e NIGHTLY='${params.NIGHTLY}' \
                    -e ROSA='${params.ROSA}' \
                    -e GODEBUG=x509ignoreCN=0 \
                    quay.io/maistra/maistra-test-tool:${MAISTRA_VERSION};
                    else echo 'Skip';
                    fi
                """,
                returnStdout: true
                ).trim()
                println OUT
            }
            }
        }
        stage("Start running a single test case"){
            dir('tests') {
            wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[var: 'OCP_CRED_PSW', password: OCP_CRED_PSW]], varMaskRegexes: []]) {
                def OUT = sh (
                script: """
                    if [ -z "${params.TEST_CASE}" ]; 
                    then echo 'Skip';
                    else docker run \
                    --name maistra-test-tool-${env.BUILD_NUMBER} \
                    -d \
                    --rm \
                    --pull always \
                    -e SAMPLEARCH='${params.OCP_SAMPLE_ARCH}' \
                    -e OCP_CRED_USR='${OCP_CRED_USR}' \
                    -e OCP_CRED_PSW='${OCP_CRED_PSW}' \
                    -e OCP_TOKEN='${params.OCP_TOKEN}' \
                    -e OCP_API_URL='${params.OCP_API_URL}' \
                    -e TEST_CASE='${params.TEST_CASE}' \
                    -e NIGHTLY='${params.NIGHTLY}' \
                    -e ROSA='${params.ROSA}' \
                    -e GODEBUG=x509ignoreCN=0 \
                    --entrypoint "../scripts/pipeline/run_one_test.sh" \
                    quay.io/maistra/maistra-test-tool:${MAISTRA_VERSION};
                    fi
                """,
                returnStdout: true
                ).trim()
                println OUT
            }
            }
        }
        stage ("Check Testing Completed") {
            def OUT = sh (
            script: """
            set +ex
            docker logs maistra-test-tool-${env.BUILD_NUMBER} | grep "#Testing Completed#"
            while [ \$? -ne 0 ]; do sleep 60; docker logs maistra-test-tool-${env.BUILD_NUMBER} | grep "#Testing Completed#"
            done
            set -ex
            """,
            returnStdout: true
            ).trim()
            println OUT
        }
        stage ("Collect logs") {
            def OUT = sh (
            script: """
            docker cp maistra-test-tool-${env.BUILD_NUMBER}:/opt/maistra-test-tool/tests/test.log .
            docker cp maistra-test-tool-${env.BUILD_NUMBER}:/opt/maistra-test-tool/tests/results.xml .
            """,
            returnStdout: true
            ).trim()
            println OUT
        }
        stage ("Validate Results") {
            junit "results.xml"
        }

    post {
        always {
            archiveArtifacts artifacts: 'test.log,results.xml'

            script {
                // we don't want to send a message for aborted jobs
                if (currentBuild.result == 'FAILURE' || currentBuild.result == 'UNSTABLE' || currentBuild.result == 'SUCCESS') {  

                    // Prepare Slack Message
                    if (currentBuild.result == 'SUCCESS') {
                            jobColor = "good"
                    } else if (currentBuild.result == 'UNSTABLE') {
                        jobColor = "warning"
                    } else {
                        jobColor = "danger"
                    }

                    // Additional information about the build
                    if (util.getWhoBuild() == "[]") {
                        executedBy = "Jenkins Trigger"
                    } else {
                        executedBy = util.whoBuild(util.getWhoBuild())
                    }                        

                    // Slack message to who ran the job
                    slackSend channel: "@lsouza", username: "jenkins", color: "${jobColor}", message: "Maistra Test Tool Job *${currentBuild.result}*\n*-Job URL:* ${env.JENKINS_URL}/job/Interop/job/interop-pipeline/${currentBuild.number}/console\n*- Triggered by:* ${triggeredBy}\n*- OCP Version:* ${ocpBuildImage}\n*- OSSM Version:* ${ossmVersion}\n"
                }
            }  
        }
    }  
}
