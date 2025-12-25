pipeline {
// 适配K8s Jenkins环境，使用动态Agent（无模板可简化为agent any）
    agent {
        kubernetes {
            // 1. 不要用label，改用inheritFrom（复用K8s插件中预定义的Pod模板）
            // 或直接在yaml中配置jnlp容器的完整通信参数
            yaml """
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      # 2. 必须保留jnlp容器（Jenkins Agent的核心通信容器）
      - name: jnlp
        image: jenkins/inbound-agent:3345.v03dee9b_f88fc-1  # 与你实际拉取的镜像一致
        args: ["\$(JENKINS_SECRET)", "\$(JENKINS_NAME)"]  # 自动注入Master地址/令牌
      # 3. 你的ssh-client容器
      - name: ssh-client
        image: alpine:3.18
        command: ['sh', '-c', 'apk add --no-cache openssh-client git && sleep infinity']
        tty: true
    """
        }
    }

    // 全局环境变量（按需修改，*为必填项）
    environment {
        // 1. 代码仓库配置（*必填）
        GIT_REPO = "https://github.com/ChenFengXiu05/RuoYi-Cloud-Plus.git"  // 代码仓库地址
        GIT_BRANCH = "2.X"  // 项目分支（如main/master/2.X）
        // 2. K8s内部节点配置（*必填）
        K8S_NODE_IP = "192.168.11.52"  // K8s内部节点IP（如节点内网IP）
        PROJECT_DIR = "/opt/project"  // 节点上项目存放目录（已提前创建）
        // 3. 凭证ID（*必填，与Jenkins中配置的一致）
        GIT_CRED_ID = "github-pw"
        SSH_CRED_ID = "k8s-node-ssh-cred"
    }

    // 流水线阶段（无Harbor相关步骤，简洁高效）
    stages {
        // 阶段1：拉取项目代码（获取docker-compose.yml及项目文件）
        stage('拉取项目代码') {
            steps {
                echo "🔍 开始拉取代码：${GIT_REPO}（分支：${GIT_BRANCH}）"
                container('ssh-client') {
                    git url: "${GIT_REPO}", credentialsId: "${GIT_CRED_ID}", branch: "${GIT_BRANCH}"
                    echo "✅ 代码拉取成功，当前工作目录：${WORKSPACE}"
                    // 验证docker-compose.yml是否存在
                    sh "ls -l ${WORKSPACE}/script/docker | grep docker-compose.yml || (echo '❌ 未找到docker-compose.yml' && exit 1)"
                }
            }
        }

        // 阶段2：同步代码到K8s内部节点（核心：将项目文件推送到目标节点）
        stage('同步代码到K8s内部节点') {
            steps {
                echo "📤 开始同步代码到K8s节点：${K8S_NODE_IP}:${PROJECT_DIR}"
                container('ssh-client') {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: "${SSH_CRED_ID}",
                        usernameVariable: "SSH_USER",
                        keyFileVariable: "SSH_PRIVATE_KEY"
                    )]) {
                        sh """
                            echo '注入的SSH用户名：\${SSH_USER}'  # 反斜杠转义，让Shell解析
                            echo '注入的私钥路径：\${SSH_PRIVATE_KEY}'
                            # 修复2：明确指定私钥路径（-i 参数），使用解析后的变量
                            ssh -i \${SSH_PRIVATE_KEY} -o StrictHostKeyChecking=no \${SSH_USER}@${K8S_NODE_IP} "mkdir -p ${PROJECT_DIR}"
                            scp -i \${SSH_PRIVATE_KEY} -o StrictHostKeyChecking=no -r \${WORKSPACE}/* \${SSH_USER}@${K8S_NODE_IP}:\${PROJECT_DIR}/
                        """
                    }
                    echo "✅ 代码同步成功，K8s节点项目目录：${PROJECT_DIR}"
                }
            }
        }

        // 阶段3：在K8s节点执行Docker Compose部署（无需Harbor，直接启动）
        stage('Docker Compose部署应用') {
            steps {
                echo "🚀 开始在K8s节点执行Docker Compose部署"
                container('ssh-client') {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: "${SSH_CRED_ID}",
                        usernameVariable: "SSH_USER",
                        keyFileVariable: "SSH_PRIVATE_KEY"
                    )]) {
                        sh """
                            # 远程连接K8s节点，执行Docker Compose命令
                            ssh -i \${SSH_PRIVATE_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${K8S_NODE_IP} << EOF
                                # 进入项目目录
                                cd ${PROJECT_DIR}/script/docker
                                # 可选：停止并删除旧容器（更新应用时用，避免缓存问题）
                                docker-compose down || true
                                # 启动/更新应用（-d后台运行，--build强制构建本地镜像，无需Harbor）
                                docker-compose up -d --build
                                # 验证容器状态
                                echo "=== 容器运行状态 ==="
                                docker-compose ps
                                echo "=== 节点Docker信息 ==="
                                docker info | grep "Server Version"
                            EOF
                        """
                    }
                    echo "✅ Docker Compose部署命令执行完成"
                }
            }
        }

        // 阶段4：验证部署结果（确保应用在K8s节点正常运行）
        stage('验证部署结果') {
            steps {
                echo "🔎 开始验证K8s节点上的应用状态"
                container('ssh-client') {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: "${SSH_CRED_ID}",
                        usernameVariable: "SSH_USER",
                        keyFileVariable: "SSH_PRIVATE_KEY"
                    )]) {
                        sh """
                            ssh -i \${SSH_PRIVATE_KEY} -o StrictHostKeyChecking=no ${SSH_USER}@${K8S_NODE_IP} << EOF
                                cd ${PROJECT_DIR}
                                # 查看应用日志（最近20行）
                                echo "=== 应用日志（最近20行） ==="
                                docker-compose logs --tail=20
                                # 查看端口映射（确认应用端口已暴露）
                                echo "=== 端口映射信息 ==="
                                docker-compose port 你的服务名 容器端口 || echo "端口查询完成"
                            EOF
                        """
                    }
                    echo "✅ 应用部署验证完成，可通过K8s节点IP:应用端口访问"
                }
            }
        }
    }

    // 流水线结束后通知
    post {
        success {
            echo "🎉 部署流程全部执行成功！"
            echo "📌 部署详情："
            echo "   - 代码仓库：${GIT_REPO}（分支：${GIT_BRANCH}）"
            echo "   - 部署节点：${K8S_NODE_IP}"
            echo "   - 项目目录：${PROJECT_DIR}"
            echo "   - 访问方式：http://${K8S_NODE_IP}:应用映射端口"
        }
        failure {
            echo "❌ 部署流程执行失败，请查看控制台日志排查问题！"
        }
    }
}
