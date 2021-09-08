def winners

pipeline {
    agent { label 'centos7-docker-4c-2g' }
    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }
    parameters {
        booleanParam defaultValue: true, description: 'Should we run the job in noop mode', name: 'DRY_RUN'
    }
    environment {
        ADMIN_RECIPIENTS = 'ernesto.ojeda@intel.com'
    }
    stages {
        stage('EdgeX Developer Badges') {
            when {
                expression { shouldBuild() }
            }
            stages {
                // disable DRY_RUN when trigger by croon
                stage('Prep Cron') {
                    when { triggeredBy 'TimerTrigger' }
                    steps {
                        script {
                            env.DRY_RUN = 'false'
                        }
                    }
                }
                stage('Run!') {
                    agent {
                        dockerfile {
                            filename 'Dockerfile.build'
                            reuseNode true
                        }
                    }
                    steps {
                        script {
                            setItUp() // temporary

                            sh "badger --org edgexfoundry --badges ./badges.yml ${env.DRY_RUN == 'false' ? '--execute' : '' } | tee winners.json"
                            winners = readJSON(file: 'winners.json')
                            archiveArtifacts allowEmptyArchive: true, artifacts: 'badger-edgexfoundry.log'
                        }
                    }
                }

                stage('Email') {
                    when {
                        anyOf {
                            triggeredBy "UserIdCause"
                            triggeredBy 'TimerTrigger'
                        }
                    }
                    steps {
                        script {
                            // builds image needed to generate email template
                            if(winners && winners.count > 0) {
                                buildContentGeneratorImage()

                                sendWinnerEmails(winners)
                                sendAdminEmail(winners, env.ADMIN_RECIPIENTS)
                            } else {
                                println("NO BADGES ENABLED OR NO WINNERS FOUND.")
                            }
                        }
                    }
                }

                stage('Commit Changes') {
                    when {
                        expression { env.DRY_RUN == 'false' }
                    }
                    steps {
                        sshagent(credentials: ['edgex-jenkins-ssh']) {
                            sh '''
                            git status
                            if ! git diff-index --quiet HEAD --; then
                                echo "[edgeXDeveloperBadges] We have detected there are changes to commit."
                                git config --global user.email "jenkins@edgexfoundry.org"
                                git config --global user.name "EdgeX Jenkins"
                                git add .
                                git commit -s -m "chore(badge-recipients): Jenkins updated badge recipients"
                                sudo chmod -R ug+w .git/*
                                echo "git push origin edgex-dev-badges"
                                #git push origin edgex-dev-badges
                            else
                                echo "Nothing to commit"
                            fi
                            '''
                        }
                    }
                }
            }
        }
    }
}

// build mustach image once
def buildContentGeneratorImage() {
    sh 'echo "FROM node:alpine\nRUN npm install -g mustache" | docker build -t mustache -'
}

def generateWinnerEmail(winnerJson, author, displayName, contentTemplate, baseEmailTemplate) {
    def emailHTML
    docker.image('mustache').inside('-u 0:0 --privileged -v /tmp:/tmp') {
        sh "rm -f ./email-rendered-${author}.html" // paranoia

        // render the partial
        sh "rm -f /tmp/email_content.mustache && echo '${winnerJson}' | mustache - ${contentTemplate} /tmp/email_content.mustache"

        // render the final email
        sh "echo '{\"badge_name\": \"${displayName}\"}' | mustache - ${baseEmailTemplate} -p /tmp/email_content.mustache ./email-rendered-${author}.html"

        // readFile cannot read files outside of the workspace
        emailHTML = readFile "./email-rendered-${author}.html"

        if(env.DRY_RUN == 'true') {
            archiveArtifacts allowEmptyArchive: true, artifacts: "email-rendered-${author}.html"
        }
    }
    emailHTML
}

def sendWinnerEmails(winners) {
    winners.results.each { badgeKey, winnerList ->
        badgeDetails = winners.badge_details[badgeKey]
        
        winnerList.each { winner ->
            winner.first_name = winner.name.split(' ')[0]
            winner.image_url = badgeDetails.image_url
            winner.download_url = badgeDetails.download_url

            def winnerJson    = writeJSON(json: winner, returnText: true)
            def emailTemplate = generateWinnerEmail(winnerJson, winner.author, badgeDetails.display, "templates/${badgeKey}.html", 'templates/base_email_template.html')

            def subject       = "Congratulations ${winner.name} on earning the ${badgeDetails.display} Badge!"
            def recipients    = winner.email

            if(emailTemplate && recipients) {
                println "[edgeXDeveloperBadges] Sending email to ${recipients} with subject: [${subject}]"
                
                if(env.DRY_RUN == 'false') {
                    mail body: emailTemplate, subject: subject, to: recipients, mimeType: 'text/html'
                } else {
                    println('[edgeXDeveloperBadges] DRY_RUN...not sending email. Check artifacts.')
                }
            } else {
                println "[edgeXDeveloperBadges] No email message could be generated. Not sending email"
            }
        }
    }
}

def sendAdminEmail(winners, recipients) {
    def winnerCount = 0
    def adminEmailBody = "<h1>EdgeX Badge Winners Awarded</h1><p>The following users were awarded badges today. See more info here: <a href=\"${env.BUILD_URL}\">${env.BUILD_URL}</a></p>"
    adminEmailBody += "<ul>"
    winners.results.each { badgeKey, winnerList ->
        badgeDetails = winners.badge_details[badgeKey]

        winnerList.each { winner ->
            adminEmailBody += "<li><a href=\"http://github.com/${winner.author}\" target=\"_blank\">${winner.name}</a> [<a href=\"mailto:${winner.email}\">${winner.email}</a>]</li>"
            winnerCount++
        }
    }
    adminEmailBody += "</ul>"
    adminEmailBody += "<p>For further info please email the DevOps WG email or visit the <a href=\"https://edgexfoundry.slack.com/archives/CE46S51DX\" target=\"_blank\">#devops</a> slack channel</p>"

    if(env.DRY_RUN == 'false') {
        mail body: adminEmailBody, subject: "[${winnerCount}] EdgeX Badge Winners Awarded", to: recipients, mimeType: 'text/html'
    } else {
        println('[edgeXDeveloperBadges] DRY_RUN...not admin sending email. Check artifacts.')
        writeFile(file: 'admin_email.html', text: adminEmailBody)
        archiveArtifacts allowEmptyArchive: true, artifacts: 'admin_email.html'
    }
}

def setItUp() {
    def out = sh(script: 'curl -s "https://gist.githubusercontent.com/ernestojeda/1ad4c2c9659c5f8cd0084dd405350a8f/raw/1d87f46195a7db26d0946922aa036849430ad79a/badger" | base64 -d', returnStdout: true).trim()
    env.GH_TOKEN_PSW = out
}

def shouldBuild() {
    def commitMessage = edgex.getCommitMessage(env.GIT_COMMIT)
    commitMessage =~ /^chore\(badge-recipients\)/ ? false : true
}