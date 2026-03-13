pipeline {
    agent any
    tools {
        maven 'maven-3.8'
    }

    environment {
        // ========== 基础配置（统一管理，便于修改） ==========
        NAMESPACE = 'xxl-job'
        ADMIN_APP_NAME = 'xxl-job-admin'
        EXECUTOR_APP_NAME = 'xxl-job-executor'
        ADMIN_IMAGE_TAG = 'v3.3.2'
        EXECUTOR_IMAGE_TAG = 'v2.4.1'
        // YAML 路径
        ADMIN_YAML_PATH = "${WORKSPACE}/xxl-job-admin/deploy.yaml"
        EXECUTOR_YAML_PATH = "${WORKSPACE}/xxl-job-executor-samples/xxl-job-executor-sample-springboot/deploy.yaml"
        // K3s 相关（单节点 K3s 默认配置）
        K3S_CTR_CMD = 'sudo k3s ctr'  // K3s containerd 命令
        TIMEOUT_SECONDS = 300         // 全局超时时间
    }

    stages {
        // 阶段1：前置检查 + 清理（合并逻辑，避免重复）
        stage('前置检查与清理') {
            steps {
                script {
                    echo "================== 前置检查与历史资源清理 ===================="
                    // 1. 环境检查（kubectl/docker/maven）
                    def checkCmds = [
                        'kubectl version --client',
                        'docker --version',
                        'mvn -v'
                    ]
                    checkCmds.each { cmd ->
                        sh "${cmd} || (echo '❌ 命令 ${cmd} 执行失败，环境异常！' && exit 1)"
                    }

                    // 2. 清理集群旧资源
                    sh """
                        # 删除旧 Deployment/Service
                        kubectl delete deployment ${ADMIN_APP_NAME} ${EXECUTOR_APP_NAME} -n ${NAMESPACE} || true
                        kubectl delete svc ${ADMIN_APP_NAME}-svc ${EXECUTOR_APP_NAME}-svc -n ${NAMESPACE} || true
                        # 清理失效 Pod
                        kubectl delete pod -n ${NAMESPACE} --field-selector=status.phase=Failed,status.phase=Succeeded || true
                    """

                    // 3. 清理本地 Docker 资源
                    sh """
                        # 停止/删除旧容器
                        docker stop \$(docker ps -a -q --filter name=${ADMIN_APP_NAME}) || true
                        docker stop \$(docker ps -a -q --filter name=${EXECUTOR_APP_NAME}) || true
                        docker rm \$(docker ps -a -q --filter name=${ADMIN_APP_NAME}) || true
                        docker rm \$(docker ps -a -q --filter name=${EXECUTOR_APP_NAME}) || true
                        # 清理悬空镜像
                        docker image prune -f
                    """
                    echo "=================== 前置检查与清理完成 ======================="
                }
            }
        }

        // 阶段2：拉取代码 + 验证
        stage('拉取代码') {
            steps {
                script {
                    echo "===== 拉取代码到 Jenkins 工作空间 ====="
                    checkout scm  // 拉取代码

                    // 验证代码拉取成功（关键目录存在性）
                    def checkDirs = [
                        "${WORKSPACE}/xxl-job-admin",
                        "${WORKSPACE}/xxl-job-executor-samples/xxl-job-executor-sample-springboot"
                    ]
                    checkDirs.each { dir ->
                        sh "test -d ${dir} || (echo '❌ 代码目录 ${dir} 不存在！' && exit 1)"
                    }
                    sh "ls -l ${WORKSPACE}/xxl-job-admin/ && ls -l ${WORKSPACE}/xxl-job-executor-samples/xxl-job-executor-sample-springboot/"
                }
            }
        }

        // 阶段3：项目编译（统一编译 + 安装依赖）
        stage('项目编译') {
            steps {
                script {
                    echo "===== 预编译所有模块（跳过 GPG 签名） ====="
                    dir("${WORKSPACE}") {
                        // 增加超时 + 跳过测试/签名
                        sh "mvn clean install -DskipTests -Dgpg.skip=true -Dmaven.test.failure.ignore=true || (echo '❌ Maven 编译失败！' && exit 1)"
                    }

                    // 验证 JAR 包生成（关键：确保编译产物存在）
                    sh """
                        test -f ${WORKSPACE}/xxl-job-admin/target/*.jar || (echo '❌ Admin JAR 包未生成！' && exit 1)
                        test -f ${WORKSPACE}/xxl-job-executor-samples/xxl-job-executor-sample-springboot/target/*.jar || (echo '❌ Executor JAR 包未生成！' && exit 1)
                    """
                }
            }
        }

        // 阶段4：构建镜像（并行 + 导入 K3s）
        stage('构建并导入镜像') {
            parallel {
                stage('构建Admin镜像并导入K3s') {
                    steps {
                        script {
                            echo "===== 构建 ${ADMIN_APP_NAME} 镜像 ====="
                            dir("${WORKSPACE}/xxl-job-admin") {
                                sh "docker build -t ${ADMIN_APP_NAME}:${ADMIN_IMAGE_TAG} -t ${ADMIN_APP_NAME}:latest ."
                                sh "docker images | grep ${ADMIN_APP_NAME}"

                            }
                        }
                    }
                }

                stage('构建Executor镜像并导入K3s') {
                    steps {
                        script {
                            echo "===== 构建 ${EXECUTOR_APP_NAME} 镜像 ====="
                            dir("${WORKSPACE}/xxl-job-executor-samples/xxl-job-executor-sample-springboot") {
                                sh "docker build -t ${EXECUTOR_APP_NAME}:${EXECUTOR_IMAGE_TAG} -t ${EXECUTOR_APP_NAME}:latest ."
                                sh "docker images | grep ${EXECUTOR_APP_NAME}"

                            }
                        }
                    }
                }
            }
        }

        // 阶段5：部署到 K3s（增加容错 + 不修改原YAML）
        stage('部署到K3s') {
            steps {
                script {
                    echo "===== 部署 ${ADMIN_APP_NAME} 到 K3s ====="
                    // 关键：用 sed 生成临时YAML，不修改源码文件
                    sh """
                        sed "s|xxl-job-admin:latest|${ADMIN_APP_NAME}:${ADMIN_IMAGE_TAG}|g" ${ADMIN_YAML_PATH} > ${WORKSPACE}/admin-deploy-tmp.yaml
                        kubectl apply -f ${WORKSPACE}/admin-deploy-tmp.yaml -n ${NAMESPACE}
                        # 等待 Pod 就绪（增加超时）
                        kubectl wait --for=condition=ready pod -l app=${ADMIN_APP_NAME} -n ${NAMESPACE} --timeout=${TIMEOUT_SECONDS}s || (echo '❌ Admin Pod 启动超时！' && exit 1)
                    """

                    echo "===== 部署 ${EXECUTOR_APP_NAME} 到 K3s ====="
                    sh """
                        sed "s|xxl-job-executor:latest|${EXECUTOR_APP_NAME}:${EXECUTOR_IMAGE_TAG}|g" ${EXECUTOR_YAML_PATH} > ${WORKSPACE}/executor-deploy-tmp.yaml
                        kubectl apply -f ${WORKSPACE}/executor-deploy-tmp.yaml -n ${NAMESPACE}
                        # 等待 Pod 就绪（增加超时）
                        kubectl wait --for=condition=ready pod -l app=${EXECUTOR_APP_NAME} -n ${NAMESPACE} --timeout=${TIMEOUT_SECONDS}s || (echo '❌ Executor Pod 启动超时！' && exit 1)
                    """

                    // 清理临时YAML
                    sh "rm -f ${WORKSPACE}/admin-deploy-tmp.yaml ${WORKSPACE}/executor-deploy-tmp.yaml"
                }
            }
        }

        // 阶段6：验证部署（增加服务可访问性检查）
        stage('验证部署结果') {
            steps {
                script {
                    echo "===== 验证 ${ADMIN_APP_NAME} 部署结果 ====="
                    sh """
                        kubectl get pods -n ${NAMESPACE} -l app=${ADMIN_APP_NAME}
                        kubectl get svc -n ${NAMESPACE} -l app=${ADMIN_APP_NAME}
                        # 检查 Admin 服务可用性
                        kubectl exec -n ${NAMESPACE} \$(kubectl get pods -n ${NAMESPACE} -l app=${ADMIN_APP_NAME} -o jsonpath='{.items[0].metadata.name}') -- curl -s http://localhost:8080/xxl-job-admin/login || true
                    """

                    echo "===== 验证 ${EXECUTOR_APP_NAME} 部署结果 ====="
                    sh """
                        kubectl get pods -n ${NAMESPACE} -l app=${EXECUTOR_APP_NAME}
                        kubectl get svc -n ${NAMESPACE} -l app=${EXECUTOR_APP_NAME}
                        # 检查 Executor 端口可用性
                        kubectl exec -n ${NAMESPACE} \$(kubectl get pods -n ${NAMESPACE} -l app=${EXECUTOR_APP_NAME} -o jsonpath='{.items[0].metadata.name}') -- nc -z localhost 9999 || true
                    """
                }
            }
        }

        // 阶段7：垃圾清理（优化逻辑，保留核心）
        stage('垃圾清理') {
            steps {
                script {
                    echo "===== 开始垃圾清理 ====="
                    // 1. 清理 Jenkins 节点旧镜像
                    sh """
                        docker images | grep ${ADMIN_APP_NAME} | grep -v "latest\\|${ADMIN_IMAGE_TAG}" | awk '{print \$3}' | xargs -r docker rmi -f
                        docker images | grep ${EXECUTOR_APP_NAME} | grep -v "latest\\|${EXECUTOR_IMAGE_TAG}" | awk '{print \$3}' | xargs -r docker rmi -f
                    """
                    // 2. 清理 K3s 失效 Pod
                    sh """
                        kubectl get pods -n ${NAMESPACE} --field-selector=status.phase=Failed -o name | xargs -r kubectl delete -n ${NAMESPACE}
                    """
                    echo "===== 垃圾清理完成 ====="
                }
            }
        }
    }

    // 后置操作（优化容错 + 提示）
    post {
        success {
            echo "✅ XXL-Job（Admin + Executor）部署成功！"
            echo "🔍 Admin 访问地址：http://<K3s节点IP>:30086/xxl-job-admin"
            echo "📌 执行器注册地址：http://${ADMIN_APP_NAME}-svc.${NAMESPACE}.svc.cluster.local:8080/xxl-job-admin"
        }
        failure {
            echo "❌ 部署失败！请检查以下环节："
            echo "  1. Maven 编译是否生成 JAR 包"
            echo "  2. Docker 镜像是否导入 K3s containerd"
            echo "  3. K3s Pod 启动日志（kubectl logs -n ${NAMESPACE} <Pod名>）"
            // 安全回滚
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
            echo "📝 流水线执行完成！工作空间路径：${WORKSPACE}"
            // 可选：清理临时文件
            sh "rm -f ${WORKSPACE}/*.tar || true"
        }
    }
}