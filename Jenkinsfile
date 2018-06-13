#!groovy

properties([
        parameters([
                string(
                        defaultValue: 'aat',
                        description: 'Environment to test',
                        name: 'ENVIRONMENT'
                )]),
        pipelineTriggers([
                [$class: 'GitHubPushTrigger', displayName: 'CCD Security tests'],
                [$class: 'hudson.triggers.TimerTrigger', spec: 'H H(3-4) * * *']
        ])
])

def env_vars = [
        'aat': [
                emGW: 'https://ccd-api-gateway-web-aat.service.core-compute-aat.internal'
        ],
        'test': [
                emGW: 'https://case-api-gateway-web.test.ccd.reform.hmcts.net/'
        ]
]

def envSpace = env_vars["${env.ENVIRONMENT}"]

node() {
    try{

        stage("Install ZAP Server") {
            sh "wget --no-verbose https://github.com/zaproxy/zaproxy/releases/download/2.7.0/ZAP_2.7.0_Crossplatform.zip"
            sh "rm -rf ZAP_2.7.0"
            sh "unzip -q ZAP_2.7.0_Crossplatform.zip"
        }

        stage("Install ZAP CLI") {
            sh "sudo pip install --upgrade zapcli"
        }

        stage("Start ZAP") {
            sh "ZAP_2.7.0/zap.sh -daemon -host 127.0.0.1 -port 8090 " +
                    "-config view.mode=attack " +
                    "-config api.disablekey=true " +
                    "-config database.recoverylog=false " +
                    "-config connection.timeoutInSecs=120 &"
            sh 'zap-cli --zap-url http://127.0.0.1 -p 8090 status -t 120'
            sh "zap-cli --zap-url http://127.0.0.1 -p 8090 open-url ${envSpace.emGW}"
        }

        stage("Zap Security Scan (active-scan)") {
            try {
                sh "zap-cli --zap-url http://127.0.0.1 -p 8090 active-scan  --scanners all --recursive ${envSpace.emGW}"
                sh 'zap-cli --zap-url http://127.0.0.1 -p 8090 report -o activescan.html -f html'
            } finally {
                publishHTML target: [
                        alwaysLinkToLastBuild: true,
                        reportDir            : ".",
                        reportFiles          : "activescan.html",
                        reportName           : "ZAP Active scan test report"
                ]
            }

        }

        stage("Zap Security Scan (spider)") {
            try {
                sh "zap-cli --zap-url http://127.0.0.1 -p 8090 spider ${envSpace.emGW}"
                sh 'zap-cli --zap-url http://127.0.0.1 -p 8090 report -o spider.html -f html'
            } finally {
                publishHTML target: [
                        alwaysLinkToLastBuild: true,
                        reportDir            : ".",
                        reportFiles          : "spider.html",
                        reportName           : "ZAP spider scan test report"
                ]
            }
        }

        stage("Zap Security Scan (ajax-spider)") {

            try {
                sh "zap-cli --zap-url http://127.0.0.1 -p 8090 ajax-spider ${envSpace.emGW}"
                sh 'zap-cli --zap-url http://127.0.0.1 -p 8090 report -o ajax-spider.html -f html'
            } finally {
                publishHTML target: [
                        alwaysLinkToLastBuild: true,
                        reportDir            : ".",
                        reportFiles          : "ajax-spider.html",
                        reportName           : "ZAP ajax spider scan test report"
                ]
            }
        }

        stage("Zap Security Scan (Alert)") {
            try{
                sh 'zap-cli --zap-url http://127.0.0.1 -p 8090 alerts -l Low'
            }catch (Exception err){
                slackSend(
                        channel: "#ccd-notifications",
                        color: 'danger',
                        message: "${env.JOB_NAME}:  <${env.RUN_DISPLAY_URL}| Security scan ${env.BUILD_DISPLAY_NAME}> is vulnerable"
                )
                System.exit(err)
            }
        }


    } catch (Exception err) {
        slackSend(
                channel: "#ccd-notifications",
                color: 'danger',
                message: "${env.JOB_NAME}:  <${env.RUN_DISPLAY_URL}| Security scan ${env.BUILD_DISPLAY_NAME}> has FAILED"
        )
        System.exit(err)
    }
}
