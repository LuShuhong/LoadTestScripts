def singleTest(n, app, target) {
    return {
        node("loadtest") {
            script {
                deleteDir()
                sh "set +x && source /home/jenkinsproduct/.bash_profile && set -x && artillery run /home/jenkinsproduct/${app}/testplan.yml -o ./${currentBuild.number}-output${n}.json -t ${target}"
                stash name: "${currentBuild.number}-${n}"
            }
        }
    }
}

parameters {
    string(
            name: 'target',
            defaultValue: 'No Target Selected',
            description: 'Target to load test')
    string(
            name: 'app',
            defaultValue: 'No App Selected',
            description: 'App to load test')
    string(
            name: 'numCPUs',
            defaultValue: '1',
            description: 'Number of CPU to be used')
    booleanParam(
            name: 'isTriggered',
            defaultValue: false,
            description: 'Has build been triggered by another job'
    )
}

pipeline {

    agent none

    options {
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
    }

    stages {
        stage('Select Config'){
            steps{
                node("master"){
                    deleteDir()
                    script{

                        if (!params.isTriggered) {
                            config = input(
                                    ok: 'Deploy',
                                    abort: 'Abort',
                                    parameters: [
                                            string(name: 'target', description: 'e.g. http://search.pl.io.thehut.local:8080'),
                                            string(name: 'app', description: 'e.g. search'),
                                            string(name: 'numCPUs', defaultValue: '1')
                                    ]
                            )
                            target = config["target"]
                            app = config["app"]
                            numCPUs = config["numCPUs"].toInteger()
                        } else {
                            target = params.target
                            app = params.app
                            numCPUs = params.numCPUs.toInteger()
                        }

                        if (numCPUs < 1 || numCPUs > 32)
                            error("${numCPUs} is invalid, numCPUs has to be between 1 and 32")

                        currentBuild.displayName = "${currentBuild.number} - ${app} x ${numCPUs}"
                    }
                }
            }
        }
        stage('Load Test') {
            steps {
                node("master") {
                    script {
                        def map = [:]
                        for (int i = 1; i <= numCPUs; i++) {
                            map[i] = singleTest(i, app, target)
                        }
                        parallel map
                    }
                }
            }
            post {
                always {
                    node("loadtest") {
                        script {
                            for (int i = 1; i <= numCPUs; i++) {
                                unstash "${currentBuild.number}-${i}"
                            }
                            archiveArtifacts "*.json"
                            deleteDir()

                        }
                    }
                }
                aborted {
                    node("loadtest") {
                        script {
                            sh "/home/jenkinsproduct/killArtillery.sh ${app}"
                        }
                    }
                }

            }
        }
    }

}
