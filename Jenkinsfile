pipeline {
    agent any

    environment {
        // 定义环境变量
        DOCKER_CREDENTIAL_ID = 'docker-hub-credentials' // Jenkins 中的凭据 ID
        COMPOSE_FILE = 'docker-compose.build.yml'
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
                sh 'docker compose -f ${COMPOSE_FILE} build --build-arg HTTP_PROXY=http://host.docker.internal:7897 --build-arg HTTPS_PROXY=http://host.docker.internal:7897'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo "正在推送到 Docker Hub..."
                // 使用 Jenkins 的凭据和 Docker 登录
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIAL_ID}", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'
                    sh 'docker compose -f ${COMPOSE_FILE} push'
                }
            }
        }

        stage('Deploy') {
            steps {
                echo "开始部署服务..."
                // 更新服务：拉取最新镜像并重启
                sh 'docker-compose pull'
                sh 'docker-compose up -d'
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
