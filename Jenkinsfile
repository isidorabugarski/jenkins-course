def output = ""

def sendEmail(jobName, jobID, jobStatus, testResults){
    echo (message: "Job ${jobName} with the ID ${jobID} returned with status ${jobStatus} and the test results were: ${testResults}")
    return 1
}

pipeline {
    agent any
    parameters {
        string defaultValue: 'pipeline_day3.zip', name: 'ARTIFACT_NAME', trim: true
        booleanParam defaultValue: false, name: 'FAIL_PIPELINE'
        booleanParam defaultValue: true, name: 'RUN_TEST'
    }
    stages {
        stage('Download'){
            steps {
                cleanWs()
                echo (message: "Stage: Download")
                dir('pipeline'){
                    git(
                        branch: 'pipeline',
                        url: 'https://github.com/KLevon/jenkins-course'
                    )
                }
                rtDownload (
                    serverId: 'Artifactory',
                    spec: '''{
                        "files": [
                        {
                          "pattern": "generic-local/libraries/printer.zip",
                          "target": "printer/"
                        }
                      ]
                    }'''
                )
                unzip (
                    zipFile: "printer/libraries/printer.zip",
                    dir: "pipeline/"
                )
            }
        }
        stage('Build'){
            steps {
                echo (message: "Stage: Build")
                dir('pipeline') {
                    bat (
                        script: """
                            Makefile.bat
                        """
                    )
                }
                withCredentials(
                    [usernamePassword(credentialsId: 'USER_CREDENTIALS', passwordVariable: 'password', usernameVariable: 'username')]
                ) {
                    
                    echo "Jenkins credentials - username: ${username} and password: ${password}"
                }
            }
        }
        stage('Tests'){
            when {
                equals expected: true,
                actual: params.RUN_TEST
            }
            steps {
                echo (message: "Stage: Tests")
                script {
                    def array = ["printer", "scanner", "main"]
                    for (element in array) {
                        output += bat (
                            script: """
                                pipeline/Tests.bat ${element}
                            """, returnStdout: true
                        ).trim()
                    }
                }
            }
        }
        stage('Dynamic'){
            when {
                branch 'feature/multi/*'
            }
            steps {
                echo (message: "Stage: Dynamic")
            }
        }
        stage('Publish'){
            steps {
                echo (message: "Stage: Publish")
                script {
                    zip (
                        zipFile: "${params.ARTIFACT_NAME}",
                        archive: true,
                        dir: "pipeline/"
                    )
                }
                rtUpload (
                    serverId: 'Artifactory',
                    spec: """{
                        "files": [
                        {
                          "pattern": "${params.ARTIFACT_NAME}",
                          "target": "generic-local/release/isidora/${env.BUILD_ID}/"
                        }
                      ]
                    }"""
                )
                script {
                    if(params.FAIL_PIPELINE == true) {
                        bat (
                            script: """
                                exit 1
                            """
                        )
                    }
                }
            }
        }
    }
    post {
        failure {
            script {
                    sendEmail(env.JOB_NAME, env.BUILD_ID, currentBuild.currentResult, output)
                }
        }
    }
}
