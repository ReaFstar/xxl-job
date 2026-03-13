pipeline {
    agent any
    tools {
        maven 'maven-3.8'  // 执行器是 SpringBoot 项目，需要 Maven 打包
    }

    environment {
        // 核心配置（统一版本和路径，适配你的项目架构）
        NAMESPACE = 'xxl-job'
        ADMIN_APP_NAME = 'xxl-job-admin'
        EXECUTOR_APP_NAME = 'xxl-job-executor'
        ADMIN_IMAGE_TAG = 'v3.3.2'  // Admin 镜像版本
        EXECUTOR_IMAGE_TAG = 'v2.4.1'  // 执行器镜像版本
        // 代码内的 YAML 路径（关键：指向拉取代码后的目录，而非宿主机固定目录）
        ADMIN_YAML_PATH = 'xxl-job-admin/deploy.yaml'
        EXECUTOR_YAML_PATH = 'xxl-job-executor-samples/xxl-job-executor-sample-springboot/deploy.yaml'
    }

    stages {
        // 阶段1：前置检查（先检查环境，再拉代码）
        stage('Pre-Check') {
            steps {
                script {
                    echo "===== 开始前置检查 ====="
                    // 1. 检查 kubectl 可用性
                    sh 'kubectl version --client || (echo "kubectl 未安装或配置错误！" && exit 1)'
                    // 2. 检查 Docker 可用性（构建镜像用）
                    sh 'docker --version || (echo "Docker 未安装！" && exit 1)'
                    // 3. 检查 Maven 可用性（执行器打包用）
                    sh 'mvn -v || (echo "Maven 配置错误！" && exit 1)'
                    // 4. 检查命名空间，不存在则创建
                    sh "kubectl create namespace ${NAMESPACE} || true"
                    echo "===== 前置检查通过 ====="
                }
            }
        }

        // 阶段2：拉取代码（前置检查通过后拉取）
        stage('拉取代码') {
            steps {
                script {
                    echo "===== 拉取代码到 Jenkins 节点 ====="
                    checkout scm  // 拉取代码到工作空间（默认路径：/var/jenkins_home/workspace/任务名/）
                    // 验证代码拉取成功（修正执行器目录路径）
                    sh "ls -l ./xxl-job-admin/ && ls -l ./xxl-job-executor-samples/xxl-job-executor-sample-springboot/"
                }
            }
        }

        // 阶段3：构建镜像（并行构建 Admin + Executor，提高效率）
        stage('预编译core模块') {
            steps {
                script {
                    echo "===== 预编译 xxl-job-core 并 install ====="
                    // 切到项目根目录
                    dir("./") {
                        // 编译并 install 所有模块到本地 Maven 仓库
                        sh "mvn clean install -pl xxl-job-core -DskipTests"
                    }
                }
            }
        }

        stage('构建镜像') {
            parallel {
                stage('构建xxl-job-admin镜像') {
                    steps {
                        script {
                            echo "===== 构建 ${ADMIN_APP_NAME} 镜像 ====="
                            dir("xxl-job-admin") {
                                sh """
                                    docker build -t ${ADMIN_APP_NAME}:${ADMIN_IMAGE_TAG} -t ${ADMIN_APP_NAME}:latest .
                                """
                                sh "docker images | grep ${ADMIN_APP_NAME}"
                            }
                        }
                    }
                }

                stage('构建xxl-job-executor镜像') {
                    steps {
                        script {
                            echo "===== 构建 ${EXECUTOR_APP_NAME} 镜像 ====="
                            dir("xxl-job-executor-samples/xxl-job-executor-sample-springboot") {
                                sh """
                                    docker build -t ${EXECUTOR_APP_NAME}:${EXECUTOR_IMAGE_TAG} -t ${EXECUTOR_APP_NAME}:latest .
                                """
                                sh "docker images | grep ${EXECUTOR_APP_NAME}"
                            }
                        }
                    }
                }
            }
        }

        // 阶段4：部署到 K3s 集群（先部署 Admin，再部署 Executor）
        stage('部署到K3s') {
            steps {
                script {
                    echo "===== 部署 ${ADMIN_APP_NAME} 到 K3s ====="
                    // 部署 Admin（替换 YAML 里的镜像标签）
                    sh """
                        sed -i "s|xxl-job-admin:latest|${ADMIN_APP_NAME}:${ADMIN_IMAGE_TAG}|g" ${ADMIN_YAML_PATH}
                        kubectl apply -f ${ADMIN_YAML_PATH} -n ${NAMESPACE}
                        # 修正：shell 注释用 # 而非 //
                        kubectl wait --for=condition=ready pod -l app=${ADMIN_APP_NAME} -n ${NAMESPACE} --timeout=300s
                    """

                    echo "===== 部署 ${EXECUTOR_APP_NAME} 到 K3s ====="
                    // 部署 Executor（替换 YAML 里的镜像标签）
                    sh """
                        sed -i "s|xxl-job-executor:latest|${EXECUTOR_APP_NAME}:${EXECUTOR_IMAGE_TAG}|g" ${EXECUTOR_YAML_PATH}
                        kubectl apply -f ${EXECUTOR_YAML_PATH} -n ${NAMESPACE}
                        # 修正：shell 注释用 # 而非 //
                        kubectl wait --for=condition=ready pod -l app=${EXECUTOR_APP_NAME} -n ${NAMESPACE} --timeout=300s
                    """
                }
            }
        }

        // 阶段5：验证部署
        stage('Verify Deployment') {
            steps {
                script {
                    echo "===== 验证 ${ADMIN_APP_NAME} 部署结果 ====="
                    sh """
                        kubectl get pods -n ${NAMESPACE} -l app=${ADMIN_APP_NAME}
                        kubectl get svc -n ${NAMESPACE} -l app=${ADMIN_APP_NAME}
                        # 修正：XXL-Job v3.3.2 无 actuator，改用登录页检查
                        kubectl exec -n ${NAMESPACE} \$(kubectl get pods -n ${NAMESPACE} -l app=${ADMIN_APP_NAME} -o jsonpath='{.items[0].metadata.name}') -- curl -s http://localhost:8080/xxl-job-admin/toLogin
                    """

                    echo "===== 验证 ${EXECUTOR_APP_NAME} 部署结果 ====="
                    sh """
                        kubectl get pods -n ${NAMESPACE} -l app=${EXECUTOR_APP_NAME}
                        kubectl get svc -n ${NAMESPACE} -l app=${EXECUTOR_APP_NAME}
                        # 检查 Executor 服务是否可访问
                        kubectl exec -n ${NAMESPACE} \$(kubectl get pods -n ${NAMESPACE} -l app=${EXECUTOR_APP_NAME} -o jsonpath='{.items[0].metadata.name}') -- nc -z localhost 9999
                    """
                }
            }
        }

        // 阶段6：垃圾清理（清理临时镜像、失效 Pod）
        stage('垃圾清理') {
            steps {
                script {
                    echo "===== 开始垃圾清理 ====="
                    // 1. 清理 Jenkins 节点上的旧镜像（保留最新标签）
                    sh """
                        # 修正：shell 注释用 # 而非 //，转义符正确
                        docker images | grep ${ADMIN_APP_NAME} | grep -v "latest\\|${ADMIN_IMAGE_TAG}" | awk '{print \$3}' | xargs -r docker rmi -f
                        docker images | grep ${EXECUTOR_APP_NAME} | grep -v "latest\\|${EXECUTOR_IMAGE_TAG}" | awk '{print \$3}' | xargs -r docker rmi -f
                    """
                    // 2. 清理 K3s 集群内的失效 Pod（Evicted/Error 状态）
                    sh """
                        kubectl get pods -n ${NAMESPACE} --field-selector=status.phase=Failed -o name | xargs -r kubectl delete -n ${NAMESPACE}
                    """
                    echo "===== 垃圾清理完成 ====="
                }
            }
        }
    }

    // 后置操作：全局结果提示 + 异常清理
    post {
        success {
            echo "✅ XXL-Job（Admin + Executor）部署成功！"
            echo "🔍 访问地址：http://<K3s节点IP>:30086/xxl-job-admin"
        }
        failure {
            echo "❌ 部署失败，请查看流水线日志排查问题！"
            // 安全回滚：先判断文件是否存在
            sh """
                if [ -f "${ADMIN_YAML_PATH}" ]; then
                    kubectl delete -f ${ADMIN_YAML_PATH} -n ${NAMESPACE} || true
                fi
                if [ -f "${EXECUTOR_YAML_PATH}" ]; then
                    kubectl delete -f ${EXECUTOR_YAML_PATH} -n ${NAMESPACE} || true
                fi
            """
        }
        always {
            echo "📝 流水线执行完成，开始清理工作空间临时文件"
            cleanWs()  // 修正：直接调用函数，无需 sh 包裹
        }
    }
}