import java.text.SimpleDateFormat

def TODAY = (new SimpleDateFormat("yyyyMMddHHmmss")).format(new Date())

pipeline {
    agent { label 'master' }
    environment {
        strDockerTag = "${TODAY}_${BUILD_ID}"
        strDockerImage = "minwoong87/cicd_guestbook:${strDockerTag}"
    }

    stages {
        stage('Checkout') {
            agent { label 'agent1' }
            steps {
                echo "Checking out code from GitHub"
                git (
                    branch: 'master',
                    url: 'https://github.com/minwoong87/guestbook.git'
                )

            }
        }
        stage('Build') {
            agent { label 'agent1' }
            steps {
                echo "Building the project"
                sh "./mvnw -Dmaven.test.failure.ignore=true clean package"

            }
            post {
                success {
                    archiveArtifacts 'target/*.jar'
                }
            }
        }
        stage('Unit Test') {
            agent { label 'agent1' }
            steps {
                echo "Running unit tests"
                sh './mvnw test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/TEST-*.xml'
                }
            }
        }
        stage('SonarQube Analysis') {
            agent { label 'agent1' }
            steps {
                echo "Performing SonarQube analysis"
                withSonarQubeEnv('SonarQube-Server') {
                    sh '''
                        ./mvnw sonar:sonar \
                        -Dsonar.projectKey=guestbook \
                    '''
                }

            }
        }
        stage('SonarQube Quality Gate') {
            agent { label 'agent1' }
            steps {
                echo "Checking SonarQube quality gate"
                timeout(time: 30, unit: 'SECONDS') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            echo "NOT OK Status: ${qg.status}"
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        } else {
                            echo "OK Status: ${qg.status}"
                        }
                    }
                }

            }
        }
        stage('Docker Image Build') {
            agent { label 'agent2' }
            steps {
                //echo "Building Docker image with tag ${TODAY}_${BUILD_ID}"
                script {
                    // oDockImage = docker.build(strDockerImage)
                    oDockImage = docker.build(strDockerImage, "--build-arg VERSION=${strDockerTag} -f Dockerfile .")
                }
            }
        }
        stage('Docker Image Push') {
            agent { label 'agent2' }
            steps {
                echo "Pushing Docker image to registry"
                script {
                    docker.withRegistry('', 'DockerHub_Credential') {
                        oDockImage.push()
                    }
                }
            }
        }
        stage('Staging Deploy') {
            agent { label 'master' }
            steps {
                echo "Deploying to staging environment"
                sshagent(credentials: ['Staging-PrivateKey']) {
                    sh "ssh -o StrictHostKeyChecking=no root@172.31.0.110 whoami"
                    sh "ssh -o StrictHostKeyChecking=no root@172.31.0.110 env"
                    sh "ssh -o StrictHostKeyChecking=no root@172.31.0.110 docker container rm -f guestbookapp"
                    sh """ssh -o StrictHostKeyChecking=no root@172.31.0.110 docker container run \
                        -d \
                        -p 38080:80 \
                        --name=guestbookapp \
                        -e MYSQL_IP=172.31.0.100 \
                        -e MYSQL_PORT=3306 \
                        -e MYSQL_DATABASE=guestbook \
                        -e MYSQL_USER=root \
                        -e MYSQL_PASSWORD=education \
                        ${strDockerImage} """

                }
            }
        }
        stage('JMeter LoadTest') {
            agent { label 'agent1' }
            steps {
                echo "Running JMeter load tests"
                sh '~/lab/sw/jmeter/bin/jmeter.sh -j jmeter.save.saveservice.output_format=xml -n -t src/main/jmx/guestbook_loadtest.jmx -l loadtest_result.jtl'
                perfReport filterRegex: '', showTrendGraphs: true, sourceDataFiles: 'loadtest_result.jtl'

            }
        }
        stage("Email Notification Test") {
            steps {
                echo "email test"
            }
            post {
                always {
                    emailext(
                        attachLog: true,
                        body: '본문',
                        compressLog: true,
                        recipientProviders: [buildUser()],
                        subject: '제목',
                        to: 'minwoong.park@mirero.co.kr'
                    )
                }
            }
        }
    }
    post {
        always {
            slackSend(
                tokenCredentialId: 'slack-token',
                channel: '#소셜',
                color: 'good',
                message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 끝났습니다. Details: (<${BUILD_URL} | here >)"
            )
        }
        success {
            slackSend(
                tokenCredentialId: 'slack-token',
                channel: '#소셜',
                color: 'good',
                message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 성공적으로 끝났습니다. Details: (<${BUILD_URL} | here >)"
            )
        }
        failure {
            slackSend(
                tokenCredentialId: 'slack-token',
                channel: '#소셜',
                color: 'danger',
                message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 실패하였습니다. Details: (<${BUILD_URL} | here >)"
            )
        }
    }
}
