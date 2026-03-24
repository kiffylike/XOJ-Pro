pipeline {
    agent any

    environment {
        // 定义环境变量
        DOCKER_CREDENTIAL_ID = 'aliyun-acr-credentials' // Jenkins 中的阿里云 ACR 凭据 ID
        DOCKER_REGISTRY = 'crpi-bf3kcemfp45x9kin.cn-guangzhou.personal.cr.aliyuncs.com'
        BUILD_COMPOSE_FILE = 'docker-compose.build.yml'

        // 远程部署目标（按实际修改）
        REMOTE_HOST = '118.145.113.165'
        REMOTE_SSH_PORT = '22'
        REMOTE_APP_DIR = '/root/GIITOJ-Deploy'
        REMOTE_SSH_CREDENTIAL_ID = 'prod-server-ssh-key' // Jenkins 中的 SSH 私钥凭据 ID
        DEPLOY_REPO_URL = 'https://github.com/kiffylike/GIITOJ-Deploy.git'
        DEPLOY_REPO_BRANCH = 'main'
    }

    stages {
        stage('Checkout') {
            steps {
                // 拉取代码
                checkout scm
                echo "代码拉取成功"
            }
        }

        stage('Build Docker Images') {
            steps {
                echo "开始构建 Docker 镜像..."
                // 注入代理，让 Docker 在打包时走你的 Clash，完美解决 npm 网络被断的问题
                sh 'docker compose -f ${BUILD_COMPOSE_FILE} build --build-arg HTTP_PROXY=http://host.docker.internal:7897 --build-arg HTTPS_PROXY=http://host.docker.internal:7897'
            }
        }

        stage('Push to Aliyun ACR') {
            steps {
                echo "正在推送到阿里云 ACR..."
                // 使用 Jenkins 的凭据和 Docker 登录
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIAL_ID}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin ${DOCKER_REGISTRY}'
                    sh 'docker compose -f ${BUILD_COMPOSE_FILE} push'
                }
            }
        }

        stage('Deploy Remote') {
            steps {
                echo "开始远程部署服务..."
                withCredentials([
                    usernamePassword(credentialsId: "${DOCKER_CREDENTIAL_ID}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME'),
                    sshUserPrivateKey(credentialsId: "${REMOTE_SSH_CREDENTIAL_ID}", keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')
                ]) {
                    sh '''
                        set -e
                        ssh -i "$SSH_KEY" -p ${REMOTE_SSH_PORT} -o StrictHostKeyChecking=no "$SSH_USER@${REMOTE_HOST}" "\
                          set -e; \
                          if [ ! -d '${REMOTE_APP_DIR}/.git' ]; then \
                            git clone --depth 1 -b ${DEPLOY_REPO_BRANCH} ${DEPLOY_REPO_URL} ${REMOTE_APP_DIR}; \
                          else \
                            cd ${REMOTE_APP_DIR} && git fetch --all --prune && git checkout ${DEPLOY_REPO_BRANCH} && git reset --hard origin/${DEPLOY_REPO_BRANCH}; \
                          fi && \
                          cd ${REMOTE_APP_DIR} && \
                          echo '$DOCKER_PASSWORD' | docker login -u '$DOCKER_USERNAME' --password-stdin ${DOCKER_REGISTRY} && \
                          docker compose -f docker-compose.yml pull oj-judge oj-backend oj-frontend && \
                          docker compose -f docker-compose.yml up -d \
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ CI/CD 流程执行成功！"
        }
        failure {
            echo "❌ CI/CD 流程执行失败，请检查日志。"
        }
        always {
            // 清理未使用的镜像以释放空间（可选）
            sh 'docker image prune -f'
            echo "执行清理工作完毕"
        }
    }
}
