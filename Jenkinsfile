pipeline {
    agent any

    options {
        timestamps() // Add timestamps to logging
        timeout(time: 12, unit: 'HOURS') // Abort pipleine

        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
        disableConcurrentBuilds()
    }
    environment {
        PATH = "/usr/local/bin:$PATH"
    }
    parameters {
        choice(name: 'TestName',
               choices: [
                   'Basic GoPath',
                   'Allocation Mode',
                   'Allocation Required',
                   'Auto GPS-SDL',
                   'Block IV Non-GPS',
                   'Specific Tasks',
                   'ALL'
               ],
               description: 'Select a Testcase to run')
    }

    stages {
        stage('Init') {
            steps {
                echo "Stage: Init"
                echo "branch=${env.BRANCH_NAME}, test=${params.TestName}"
            }
        }
        stage('Common Config') {
            steps {
                echo 'Stage: Common Config'
                // Checkout repo with common config files/scripts to 'common' folder
                checkout([$class: 'GitSCM',
                          branches: [[name: '*/main']],
                          doGenerateSubmoduleConfigurations: false,
                          extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'common']],
                          submoduleCfg: [], userRemoteConfigs: [[url: 'file:///home/jenkins/gitrepos/cicd-common']]
                         ])
            }
        }
        stage('Ping Servers') {
            steps {
                echo 'Stage: Ping Servers'
                dir('common') {
                    sh '''
                    ./pingAll.sh
                    '''
                }
            }
        }
        stage('Get Testcases') {
            options {
                timeout(time: 1, unit: 'MINUTES')
            }
            steps {
                echo 'Stage: Get Testcases'
                dir('common') {
                    sh '''
                    ./tm_get_testcases.sh
                    cat ./tomahawk-tpid.txt
                    cat ./tomahawk-tcid.txt
                    '''
                }
            }
        }
        stage('Test Manager Test') {
            steps {
                echo "Stage: Test Manager Test"
                dir('common') {
                    script {
                        if ("ALL" == params.TestName) {
                            def cases = readFile("tomahawk-tcid.txt").split("\\r?\\n");
                            cases.each { String testcase_id ->
                                echo "tcid: ${testcase_id}"
                                testcase_id = testcase_id.split(":")[0]
                                build(job: '/RunTestcaseId/main', parameters: [string(name: 'testcase_id', value: "${testcase_id}")], wait: true)
                            }
                        }
                        else {
                            echo "tcname: \'${params.TestName}\'"
                            def tcid = sh(script: "grep \"${params.TestName}\" tomahawk-tcid.txt | awk -F\':\' \'{print \$1}\'", returnStdout: true).trim()
                            echo "tcid: ${tcid}"
                            build(job: '/RunTestcaseId/main', parameters: [string(name: 'testcase_id', value: "${tcid}")], wait: true)
                        }
                        
                        // Transfer DX data from TTWCS to AM
                        ./transfer_data_to_am.sh
                        
                        // Start Analysis Job
                        build(job: '/AnalysisMgr/main', wait: true)
                    }
                }
            }
        }
        /*
        stage('Start Analysis') {
            steps {
                echo "Stage: Start Analysis"
                //build(job: '/AnalysisMgr/main', wait: true)
            }
        }
        */
        stage('Cleanup') {
            steps {
                echo "Stage: Cleanup"
                // deleteDir()
            }
        }
    }
    post {
        always {
            echo "post/always"
            deleteDir()
        }
        success {
            echo "post/success"
        }
        failure {
            echo "post/failure"
        }
    }
}
