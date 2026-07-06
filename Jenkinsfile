pipeline {
    agent any

    parameters {
        string(name: 'TARGET_ENV', defaultValue: 'staging', description: '배포 대상 환경 (staging/production)')
    }

    environment {
        AWX_URL = 'http://192.168.192.132'              // AWX 서버 주소로 수정
        JOB_TEMPLATE_ID = '15'          // AWX Job Template 상세 URL에서 확인한 숫자
        DEST_PATH = '/tmp/config.conf'      // 파일이 배포될 서버 내 경로
    }

    stages {
        stage('Checkout & 버전 추출') {
            steps {
                checkout scm
                script {
                    env.FILE_VERSION = sh(
                        script: "git describe --tags --exact-match 2>/dev/null | sed 's/^v//' || echo ''",
                        returnStdout: true
                    ).trim()

                    if (env.FILE_VERSION == '') {
                        echo "태그 없는 push입니다. 파일 배포는 건너뜁니다."
                    } else {
                        echo "배포 대상 파일 버전: ${env.FILE_VERSION}"
                    }
                }
            }
        }

        stage('운영 배포 승인') {
            when {
                allOf {
                    expression { env.FILE_VERSION != '' }
                    expression { params.TARGET_ENV == 'production' }
                }
            }
            steps {
                input message: "파일 버전 ${env.FILE_VERSION}을(를) production에 배포할까요?"
            }
        }

        stage('AWX Job Template 실행') {
            when {
                expression { env.FILE_VERSION != '' }
            }
            steps {
                withCredentials([string(credentialsId: 'awx-api-token', variable: 'AWX_TOKEN')]) {
                    sh '''
                        RESPONSE=$(curl -s -k -X POST \
                          -H "Authorization: Bearer $AWX_TOKEN" \
                          -H "Content-Type: application/json" \
                          -d "{\\"extra_vars\\": {\\"file_version\\": \\"${FILE_VERSION}\\", \\"target_group\\": \\"${TARGET_ENV}\\", \\"dest_path\\": \\"${DEST_PATH}\\"}}" \
                          "$AWX_URL/api/v2/job_templates/$JOB_TEMPLATE_ID/launch/")

                        echo "AWX 응답: $RESPONSE"

                        JOB_ID=$(echo "$RESPONSE" | python3 -c "import sys, json; print(json.load(sys.stdin)['job'])")
                        echo "생성된 AWX Job ID: $JOB_ID"
                        echo "$JOB_ID" > job_id.txt
                    '''
                }
            }
        }

        stage('배포 결과 대기') {
            when {
                expression { env.FILE_VERSION != '' }
            }
            steps {
                withCredentials([string(credentialsId: 'awx-api-token', variable: 'AWX_TOKEN')]) {
                    sh '''
                        JOB_ID=$(cat job_id.txt)
                        STATUS="pending"

                        while [ "$STATUS" != "successful" ] && [ "$STATUS" != "failed" ] && [ "$STATUS" != "error" ]; do
                          sleep 5
                          STATUS=$(curl -s -k -H "Authorization: Bearer $AWX_TOKEN" \
                            "$AWX_URL/api/v2/jobs/$JOB_ID/" | python3 -c "import sys, json; print(json.load(sys.stdin)['status'])")
                          echo "현재 상태: $STATUS"
                        done

                        if [ "$STATUS" != "successful" ]; then
                          echo "파일 배포 실패: $STATUS"
                          exit 1
                        fi
                        echo "파일 배포 성공!"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "파일 버전 ${env.FILE_VERSION} 배포 완료 (대상: ${params.TARGET_ENV})"
        }
        failure {
            echo "파일 버전 ${env.FILE_VERSION} 배포 실패 (대상: ${params.TARGET_ENV})"
        }
    }
}