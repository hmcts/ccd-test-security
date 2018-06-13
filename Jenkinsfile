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

        stage("Start ZAP") {
            sh "/usr/share/owasp-zap/zap.sh -daemon -host 127.0.0.1 -port 8090 " +
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
                        message: "${env.JOB_NAME}:  <${env.BUILD_URL}console| Security scan ${env.BUILD_DISPLAY_NAME}> is vunrable"
                )
            }
        }


    } catch (Exception err) {
        slackSend(
                channel: "#ccd-notifications",
                color: 'danger',
                message: "${env.JOB_NAME}:  <${env.BUILD_URL}console| Security scan ${env.BUILD_DISPLAY_NAME}> has FAILED"
        )
    }
}
