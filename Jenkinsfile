pipeline {
    agent any

    tools {
        nodejs 'NodeJS_24.8'
    }

    environment {
        PROJECT_NAME = 'tungnn-workshop2'
        LOCAL_PATH = '/usr/share/nginx/html/jenkins'
        PRIVATE_FOLDER = 'tungnn2'
        LOCAL_CONTAINER = 'remote-host'
        LOCAL_APP_PATH = 'workshop2/web-performance-project1-initial'
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
                    def releaseDate = sh(script: 'date +%Y%m%d', returnStdout: true).trim()

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
                        sh """
                            npm install -g firebase-tools
                            firebase deploy --project your-firebase-project-id --only hosting
                        """
                    }
                    else if (params.DEPLOY_ENV == 'remote') {
                        sh """
                            ssh remote_user@remote_host 'mkdir -p /usr/share/nginx/html/jenkins/deploy/${BUILD_ID}'
                            scp -r web-performance-project1-initial/* remote_user@remote_host:/usr/share/nginx/html/jenkins/deploy/${BUILD_ID}/
                            ssh remote_user@remote_host 'rm -f /usr/share/nginx/html/jenkins/deploy/current && ln -s /usr/share/nginx/html/jenkins/deploy/${BUILD_ID} /usr/share/nginx/html/jenkins/deploy/current'
                        """
                    }
                }
            }
        }
    }
}
