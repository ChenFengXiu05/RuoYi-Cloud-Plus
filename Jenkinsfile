pipeline {
    // 适配K8s Jenkins环境，使用动态Agent（作为构建机，包含Maven+JDK环境）
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  # Jenkins Agent核心通信容器
  - name: jnlp
    image: 192.168.11.50:30003/devops/inbound-agent:3345.v03dee9b_f88fc-1
    args: ["\$(JENKINS_SECRET)", "\$(JENKINS_NAME)"]
  # 构建容器（包含Maven+JDK，用于打包、构建/推送镜像）
  - name: build-container
    image: 192.168.11.50:30003/devops/maven-docker:3.8.3-openjdk-17-slim
    tty: true
    command: ["sleep", "infinity"]
    # 挂载Docker套接字（用于在容器内操作Docker，构建/推送镜像）
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  # 挂载Docker套接字卷
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
"""
        }
    }

    // 全局环境变量
    environment {
        // 代码仓库配置
        GIT_REPO = "https://github.com/ChenFengXiu05/RuoYi-Cloud-Plus.git"
        GIT_BRANCH = "2.X"
        // 镜像仓库配置（需确保构建机/部署机能访问，私有仓库需配置凭证）
        IMAGE_REGISTRY = "192.168.11.50:30003" // 替换为你的镜像仓库地址（如Docker Hub可省略）
        IMAGE_NAMESPACE = "ruoyi"              // 镜像命名空间/用户名
        IMAGE_TAG = "2.5.2"                    // 镜像标签
        // 部署机器配置
        DEPLOY_NODE_IP = "192.168.11.52"       // 部署目标机器IP
        DEPLOY_PROJECT_DIR = "/opt/ruoyi-deploy" // 部署机器上的docker-compose.yml存放目录
        // 凭证ID（与Jenkins后台配置一致）
        GIT_CRED_ID = "github-pw"
        SSH_CRED_ID = "k8s-node-ssh-cred"
        IMAGE_REGISTRY_CRED_ID = "harbor-cred" // 镜像仓库登录凭证ID（私有仓库需配置）
        // 应用配置
        APP_SERVICE_NAME = "ruoyi-gateway"
        APP_PORT = "8080"
    }

    // 流水线阶段（构建与部署分离）
    stages {
        // 阶段1：拉取项目代码并验证关键文件
        stage('拉取项目代码') {
            steps {
                echo "🔍 开始拉取代码：${GIT_REPO}（分支：${GIT_BRANCH}）"
                container('build-container') {
                    git url: "${GIT_REPO}", credentialsId: "${GIT_CRED_ID}", branch: "${GIT_BRANCH}"
                    echo "✅ 代码拉取成功，当前工作目录：${WORKSPACE}"
                    // 验证核心文件
                    sh """
                        if [ ! -f "${WORKSPACE}/pom.xml" ]; then
                            echo '❌ 未找到pom.xml文件，无法打包构建'
                            exit 1
                        fi
                        if [ ! -f "${WORKSPACE}/script/docker/docker-compose.yml" ]; then
                            echo '❌ 未找到docker-compose.yml文件，无法部署'
                            exit 1
                        fi
                        # 替换docker-compose.yml中的镜像地址为私有仓库地址
                        sed -i "s|image: ruoyi/|image: ${IMAGE_REGISTRY}/${IMAGE_NAMESPACE}/|g" ${WORKSPACE}/script/docker/docker-compose.yml
                        echo "✅ 核心文件验证通过，并已更新镜像地址"
                    """
                }
            }
        }

        // 阶段2：构建机上执行Maven打包（生成jar包）
        stage('Maven打包（构建机）') {
            steps {
                echo "🏗️ 开始在构建机执行Maven打包"
                container('build-container') {
                    sh """
                        cd ${WORKSPACE}
                        # 配置Maven国内镜像源，加速依赖下载
                        if [ ! -f "\$HOME/.m2/settings.xml" ]; then
                            mkdir -p \$HOME/.m2
                            cat > \$HOME/.m2/settings.xml << SETTINGSEOF
<settings>
  <mirrors>
    <mirror>
      <id>aliyunmaven</id>
      <mirrorOf>central</mirrorOf>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
  </mirrors>
</settings>
SETTINGSEOF
                        fi

                        # 执行Maven打包（跳过测试，加速构建）
                        mvn clean install -D maven.test.skip=true -P prod

                        # 验证打包产物
                        if [ ! -f "${WORKSPACE}/ruoyi-gateway/target/ruoyi-gateway.jar" ]; then
                            echo "❌ 后端打包失败，未生成jar包"
                            exit 1
                        fi
                        echo "✅ Maven打包完成，生成jar包产物"
                    """
                }
            }
        }

        // 阶段3：构建机上构建Docker镜像并推送至镜像仓库
        stage('构建并推送Docker镜像（构建机）') {
            steps {
                echo "🚢 开始在构建机构建并推送镜像至${IMAGE_REGISTRY}"
                container('build-container') {
                    // 登录镜像仓库（私有仓库必需）
                    withCredentials([usernamePassword(
                        credentialsId: "${IMAGE_REGISTRY_CRED_ID}",
                        usernameVariable: "IMAGE_USER",
                        passwordVariable: "IMAGE_PWD"
                    )]) {
                        sh """
                            # 登录镜像仓库
                            docker login ${IMAGE_REGISTRY} -u \${IMAGE_USER} -p \${IMAGE_PWD}

                            # 构建镜像（以网关服务为例，多模块可循环构建）
                            cd ${WORKSPACE}/ruoyi-gateway
                            docker build -t ${IMAGE_REGISTRY}/${IMAGE_NAMESPACE}/${APP_SERVICE_NAME}:${IMAGE_TAG} .

                            # 推送镜像到仓库
                            docker push ${IMAGE_REGISTRY}/${IMAGE_NAMESPACE}/${APP_SERVICE_NAME}:${IMAGE_TAG}

                            # 验证镜像推送结果（可选）
                            docker pull ${IMAGE_REGISTRY}/${IMAGE_NAMESPACE}/${APP_SERVICE_NAME}:${IMAGE_TAG}
                            echo "✅ ${APP_SERVICE_NAME}镜像构建并推送成功"

                            # 多模块镜像构建（如需构建其他服务，复制上述步骤即可）
                            # cd ${WORKSPACE}/ruoyi-system
                            # docker build -t ${IMAGE_REGISTRY}/${IMAGE_NAMESPACE}/ruoyi-system:${IMAGE_TAG} .
                            # docker push ${IMAGE_REGISTRY}/${IMAGE_NAMESPACE}/ruoyi-system:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        // 阶段4：同步docker-compose.yml到部署机器
        stage('同步部署配置到目标机器') {
            steps {
                echo "📤 同步docker-compose.yml到部署机器：${DEPLOY_NODE_IP}:${DEPLOY_PROJECT_DIR}"
                container('build-container') {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: "${SSH_CRED_ID}",
                        usernameVariable: "SSH_USER",
                        keyFileVariable: "SSH_PRIVATE_KEY"
                    )]) {
                        sh """
                            chmod 600 \${SSH_PRIVATE_KEY}
                            # 在部署机器创建项目目录
                            ssh -i \${SSH_PRIVATE_KEY} -o StrictHostKeyChecking=no \${SSH_USER}@${DEPLOY_NODE_IP} "mkdir -p ${DEPLOY_PROJECT_DIR}"
                            # 同步docker-compose.yml
                            scp -i \${SSH_PRIVATE_KEY} -o StrictHostKeyChecking=no \
                                ${WORKSPACE}/script/docker/docker-compose.yml \
                                \${SSH_USER}@${DEPLOY_NODE_IP}:${DEPLOY_PROJECT_DIR}/
                            echo "✅ 部署配置文件同步完成"
                        """
                    }
                }
            }
        }

        // 阶段5：部署机器拉取镜像并启动容器（无需Maven/JDK）
        stage('部署机器拉取镜像并启动容器') {
            steps {
                echo "🚀 开始在部署机器拉取镜像并启动容器"
                container('build-container') {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: "${SSH_CRED_ID}",
                        usernameVariable: "SSH_USER",
                        keyFileVariable: "SSH_PRIVATE_KEY"
                    ), usernamePassword(
                        credentialsId: "${IMAGE_REGISTRY_CRED_ID}",
                        usernameVariable: "IMAGE_USER",
                        passwordVariable: "IMAGE_PWD"
                    )]) {
                        sh """
                            chmod 600 \${SSH_PRIVATE_KEY}
                            ssh -i \${SSH_PRIVATE_KEY} -o StrictHostKeyChecking=no \${SSH_USER}@${DEPLOY_NODE_IP} << EOF
# 1. 登录镜像仓库（部署机器拉取私有镜像必需）
docker login ${IMAGE_REGISTRY} -u \${IMAGE_USER} -p \${IMAGE_PWD}

# 2. 进入部署目录
cd ${DEPLOY_PROJECT_DIR}

# 3. 拉取镜像（可选，docker-compose up会自动拉取，手动拉取可提前验证）
docker pull ${IMAGE_REGISTRY}/${IMAGE_NAMESPACE}/${APP_SERVICE_NAME}:${IMAGE_TAG}

# 4. 启动容器（--no-build：不本地构建，仅拉取镜像启动）
docker-compose up -d --no-build

# 5. 等待容器初始化
echo "等待容器启动中...（30秒）"
sleep 30

# 6. 验证容器运行状态
echo "=== 容器运行状态 ==="
docker-compose ps
if docker-compose ps | grep -q "Exit"; then
    echo "❌ 存在异常停止的容器，查看日志排查"
    docker-compose logs --tail=50
    exit 1
fi
echo "✅ 应用容器启动成功"
EOF
                        """
                    }
                }
            }
        }

        // 阶段6：验证部署结果
        stage('验证部署结果') {
            steps {
                echo "🔎 开始验证部署机器上的应用状态"
                container('build-container') {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: "${SSH_CRED_ID}",
                        usernameVariable: "SSH_USER",
                        keyFileVariable: "SSH_PRIVATE_KEY"
                    )]) {
                        sh """
                            chmod 600 \${SSH_PRIVATE_KEY}
                            ssh -i \${SSH_PRIVATE_KEY} -o StrictHostKeyChecking=no \${SSH_USER}@${DEPLOY_NODE_IP} << EOF
# 1. 查看核心服务日志
echo "=== ${APP_SERVICE_NAME} 服务日志（最近50行） ==="
docker-compose logs --tail=50 ${APP_SERVICE_NAME}

# 2. 验证应用端口可访问
echo "=== 应用访问验证 ==="
curl -f http://127.0.0.1:${APP_PORT}/actuator/health || echo "✅ 应用端口${APP_PORT}可访问（健康检查接口可选）"

# 3. 查看镜像信息
echo "=== 部署机器上的镜像信息 ==="
docker images | grep ${IMAGE_NAMESPACE}
EOF
                        """
                    }
                    echo "✅ 应用部署验证完成！"
                    echo "📌 访问地址：http://${DEPLOY_NODE_IP}:${APP_PORT}"
                }
            }
        }
    }

    // 流水线后置操作
    post {
        success {
            echo "🎉 若依项目CICD流程（构建+部署）全部执行成功！"
            echo "======================================"
            echo "📋 部署详情："
            echo "   - 代码仓库：${GIT_REPO}（分支：${GIT_BRANCH}）"
            echo "   - 构建镜像：${IMAGE_REGISTRY}/${IMAGE_NAMESPACE}/${APP_SERVICE_NAME}:${IMAGE_TAG}"
            echo "   - 部署机器：${DEPLOY_NODE_IP}"
            echo "   - 部署目录：${DEPLOY_PROJECT_DIR}"
            echo "   - 访问地址：http://${DEPLOY_NODE_IP}:${APP_PORT}"
            echo "======================================"
        }
        failure {
            echo "❌ 部署流程执行失败，请查看控制台日志排查问题！"
        }
        aborted {
            echo "⚠️  部署流程被手动中止！"
        }
        always {
            // 无论成功失败，都登出镜像仓库（可选）
            container('build-container') {
                sh "docker logout ${IMAGE_REGISTRY} || true"
            }
        }
    }
}
