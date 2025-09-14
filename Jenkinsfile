pipeline {
    agent any

    tools {
        nodejs 'NodeJS_24.8'
    }

    environment {
        PROJECT_NAME = 'tungnn-workshop2'
        LOCAL_PATH = '/usr/share/nginx/html/jenkins'
        REMOTE_PATH = '/usr/share/nginx/html/jenkins'
        REMOTE_USER = "newbie"
        REMOTE_HOST = "118.69.34.46"
        REMOTE_PORT = "3334"
        PRIVATE_FOLDER = 'tungnn2'
        LOCAL_CONTAINER = 'remote-host'
    }

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['local', 'firebase', 'remote'],
            description: 'Chọn môi trường để deploy'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                echo '*****CHECKOUT STAGE*****'
                checkout scm
            }
        }
        stage('Build') {
            steps {
                echo '*****BUILD STAGE*****'
                sh 'npm install'
            }
        }
        stage('Lint/Test') {
            steps {
                echo '*****TEST STAGE*****'
                sh 'npm run test:ci'
            }
        }
        stage('Deploy') {
            steps {
                echo '*****DEPLOY STAGE*****'
                script {
                    def releaseDate = sh(script: 'date +%Y%m%d%H%M%S', returnStdout: true).trim()

                    if (params.DEPLOY_ENV == 'local') {
                        def releaseDir = "${LOCAL_PATH}/${PRIVATE_FOLDER}/deploy/${releaseDate}"
                        echo '*****DEPLOY LOCAL*****'
                        sh """
                            # Tạo thư mục release nếu chưa tồn tại
                            docker exec ${LOCAL_CONTAINER} mkdir -p ${releaseDir}
                            # Copy các file cần thiết vào thư mục release
                            docker cp index.html ${LOCAL_CONTAINER}:${releaseDir}/
                            docker cp 404.html ${LOCAL_CONTAINER}:${releaseDir}/
                            docker cp css ${LOCAL_CONTAINER}:${releaseDir}/
                            docker cp js ${LOCAL_CONTAINER}:${releaseDir}/
                            docker cp images ${LOCAL_CONTAINER}:${releaseDir}/

                            # Cập nhật liên kết 'current' và xóa các bản phát hành cũ hơn
                            docker exec ${LOCAL_CONTAINER} bash -c "
                                cd ${LOCAL_PATH}/${PRIVATE_FOLDER}/deploy
                                rm -f current
                                ln -s ${releaseDate} current
                                ls -1t | grep -E '^[0-9]{8}\$' | tail -n +6 | xargs -r rm -rf
                            "
                        """
                    }
                    else if (params.DEPLOY_ENV == 'firebase') {
                        echo "*****DEPLOY FIREBASE*****"
                        withCredentials([file(credentialsId: 'adc-service', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                            sh '''
                                # Set environment variable for Application Default Credentials
                                export GOOGLE_APPLICATION_CREDENTIALS=$GOOGLE_APPLICATION_CREDENTIALS

                                # Deploy using ADC (no token needed)
                                firebase deploy --only hosting --project=${PROJECT_NAME}
                            '''
                        }
                    }
                    else if (params.DEPLOY_ENV == 'remote') {
                        def releaseDir = "${REMOTE_PATH}/${PRIVATE_FOLDER}/deploy/${releaseDate}"
                        echo '*****DEPLOY REMOTE*****'
                        sshagent (credentials: ['WORKSHOP_SSH_KEY']) {
                            sh """
                                ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} -p ${REMOTE_PORT} "
                                    mkdir -p ${releaseDir}
                                "

                                # Copy các file cần thiết vào thư mục release
                                scp -o StrictHostKeyChecking=no -P ${REMOTE_PORT} index.html ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/
                                scp -o StrictHostKeyChecking=no -P ${REMOTE_PORT} 404.html ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/
                                scp -o StrictHostKeyChecking=no -P ${REMOTE_PORT} -r css ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/
                                scp -o StrictHostKeyChecking=no -P ${REMOTE_PORT} -r js ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/
                                scp -o StrictHostKeyChecking=no -P ${REMOTE_PORT} -r images ${REMOTE_USER}@${REMOTE_HOST}:${releaseDir}/

                                # Cập nhật liên kết 'current' và xóa các bản phát hành cũ hơn
                                ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} -p ${REMOTE_PORT} "
                                    cd ${REMOTE_PATH}/${PRIVATE_FOLDER}/deploy
                                    rm -f current
                                    ln -s ${releaseDate} current
                                    ls -1tr | head -n -6 | xargs rm -rf --
                                "
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def releaseDate = sh(script: 'date +%Y%m%d%H%M%S', returnStdout: true).trim()

                def envMessage = ""
                if (params.DEPLOY_ENV == 'local') {
                    envMessage = "• *Local Container:* Files deployed to ${LOCAL_CONTAINER}:${LOCAL_PATH}/${PRIVATE_FOLDER}/deploy/current/"
                } else if (params.DEPLOY_ENV == 'firebase') {
                    envMessage = "• *Firebase:* https://tungnn-workshop2.web.app"
                } else if (params.DEPLOY_ENV == 'remote') {
                    envMessage = "• *Remote:* http://${REMOTE_HOST}/jenkins/${PRIVATE_FOLDER}/deploy/current/"
                }

                slackSend(
                    channel: '#lnd-2025-workshop',
                    color: 'good',
                    message: ":white_check_mark: *SUCCESS* - Workshop2-tungnn deployment completed!\n" +
                            "• *User:* ${env.BUILD_USER ?: 'System'}\n" +
                            "• *Job:* ${env.JOB_NAME}\n" +
                            "• *Build:* #${env.BUILD_NUMBER}\n" +
                            "• *Release:* ${releaseDate}\n" +
                            "${envMessage}"
                )
            }
        }

        failure {
            echo "=== DEPLOY FAILED ==="
            slackSend(
                channel: '#lnd-2025-workshop',
                color: 'danger',
                message: ":x: *FAILED* - Workshop2-tungnn deployment failed!\n" +
                        "• *User:* ${env.BUILD_USER ?: 'System'}\n" +
                        "• *Job:* ${env.JOB_NAME}\n" +
                        "• *Build:* #${env.BUILD_NUMBER}\n" +
                        "• *Error:* Check Jenkins console for details"
            )
        }

        always {
            echo "=== CLEANUP ==="
            cleanWs()
        }
    }
}
