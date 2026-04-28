```groovy
pipeline {
    agent {
        label 'built-in'
    }
    tools {
        jdk ''
    }
    environment {
        //容器名
        CONTAINER_NAME = ''
        //项目代码地址   
        PROJECT_GIT_URL = ''
        //配置文件地址      
        CONFIG_GIT_URL = ''
        //docker registry凭证
        DOCKER_REGISTER_CREDS = credentials('registry_prod')
        //Docker仓库地址                     
        DOCKER_REGISTRY = "registry.cn-beijing.aliyuncs.com"      
        //命名空间                   
        DOCKER_NAMESPACE = ""                                              
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${CONTAINER_NAME}"
    }
    stages {
        stage('get timestamp') {
            steps {
                script {
                    def timestamp = sh(returnStdout: true, script: 'date +%Y%m%d%H%M%S').trim()
                    env.TIMESTAMP = timestamp
                }
            }
        }
        stage('fetch code') {
            steps {
                echo '----------Fetch code----------'
                git branch: 'master', credentialsId: 'git-credentials-backend', url: PROJECT_GIT_URL
            }
        }
        stage('fetch config') {
            steps {
                dir('../jenkinsfile'){
                    echo '----------Fetch config----------'
                    git branch: 'master', credentialsId: 'git-credentials-backend', url: CONFIG_GIT_URL
                }
            }
        }
        stage('maven build'){
            steps {
                sh '/usr/local/maven3/bin/mvn -v'
                sh '/usr/local/maven3/bin/mvn clean package -Dmaven.test.skip=true -U'
            }
        }
        stage('docker build'){
            steps {
                sh """
                    echo "当前时间为: ${env.TIMESTAMP}"
                    docker build -f ./Dockerfile -t ${DOCKER_IMAGE}:${env.TIMESTAMP} .
                    docker tag ${DOCKER_IMAGE}:${env.TIMESTAMP} ${DOCKER_IMAGE}:latest
                """
            }
        }
        stage('image push'){
            steps {
                sh """
                    docker login -u ${DOCKER_REGISTER_CREDS_USR} -p=${DOCKER_REGISTER_CREDS_PSW} ${DOCKER_REGISTRY}
                    docker push ${DOCKER_IMAGE}:${env.TIMESTAMP}
                    docker push ${DOCKER_IMAGE}:latest
                    docker rmi ${DOCKER_IMAGE}:${env.TIMESTAMP}
                    docker rmi ${DOCKER_IMAGE}:latest
                """
            }
        }
        stage('publish cn on xxx'){
            steps {
                script {
                    // 保存当前容器镜像作为回滚镜像
                    def runningContainer = sh(returnStdout: true, script: "ssh -t root@xxx docker ps -a -q -f name=${CONTAINER_NAME}").trim()
                    echo runningContainer
                    if (runningContainer) {
                        env.PREVIOUS_IMAGE = sh(returnStdout: true, script: "ssh -t root@xxx docker inspect ${CONTAINER_NAME} --format '{{.Config.Image}}'").trim()
                    }
                    echo
                    echo '旧镜像为'+env.PREVIOUS_IMAGE
                }
                // 执行部署
                sh """
                ssh -t root@xxx /bin/bash -c '
                CONTAINER_NAME="sk-biz-community"
                HOST_IP=\$(hostname -I|cut -d " " -f 1)
                curl -X PUT "http://xxxxx:2001/eureka/apps/XXX/$HOST_IP:xxx:10000/status?value=DOWN" -H "Accept: application/json"
                docker login -u ${DOCKER_REGISTER_CREDS_USR} -p=${DOCKER_REGISTER_CREDS_PSW} ${DOCKER_REGISTRY}
                docker pull ${DOCKER_IMAGE}:${env.TIMESTAMP}
                sleep 60s
                if docker ps -a -q -f name=\$CONTAINER_NAME | grep -q . ; then
                        echo "stop and rm container now"
                        docker stop \$CONTAINER_NAME
                        docker rm \$CONTAINER_NAME
                else
                        echo "container not found"
                fi
                curl -iX DELETE "http://xxxxx:2001/eureka/apps/SK-BIZ-COMMUNITY/\$HOST_IP:xxx:10000" -H "Accept: application/json"
                docker run --name=xxx -d -p xxx:xxx ${DOCKER_IMAGE}:${env.TIMESTAMP}
                sleep 30s
                curl -sSL http://xxx:10000/actuator/health
                '
                """
                try {
                    input message: '是否继续部署到后续服务器？', ok: '继续', submitterParameter: 'APPROVER'
                    echo "用户选择继续，执行后续部署"
                } catch (err) {
                    echo "用户中止部署，执行回滚操作"
                    if (env.PREVIOUS_IMAGE) {
                        sh """
                        ssh root@xxx /bin/bash -c '
                        CONTAINER_NAME=""
                        docker rm -f \$CONTAINER_NAME
                        echo "回滚到之前的镜像: ${env.PREVIOUS_IMAGE}"
                        docker run --name=xxx -d -p xxx:xxx ${env.PREVIOUS_IMAGE}
                        sleep 30s
                        curl -sSL http://xxxxx/actuator/health
                        '
                        """
                    } else {
                        echo "没有可用的回滚镜像，跳过回滚"
                    }
                    error "部署被用户中止，已执行回滚"
                }
            }
        }
        stage('publish cn on xxx'){
            steps {
                sh """

                """
            }
        }
    }
}
```

- **镜像版本控制**：通过时间戳确保每次构建生成唯一镜像。
- **安全认证**：使用 Jenkins 凭证管理 Git 和 Docker 仓库的账号密码。
- **灰度与回滚**：先停服再更新，保留旧镜像快速回滚。
- **人工干预**：关键步骤需人工确认，避免全自动部署的风险

## Jenkins Pipeline 通过接口获取镜像tag并应用

```groovy
pipeline {
    agent {
        label 'built-in'
    }
    environment {
        SERVICE_NAME = ""
        DOCKER_REGISTER_CREDS = credentials('registry_prod')
        DOCKER_REGISTRY = "registry-vpc.cn-beijing.aliyuncs.com"
        DOCKER_OUTSIDE_REGISTRY = "registry.cn-beijing.aliyuncs.com"
        DOCKER_NAMESPACE = ""
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${CONTAINER_NAME}"
        DOCKER_OUTSIDE_IMAGE = "${DOCKER_OUTSIDE_REGISTRY}/${DOCKER_NAMESPACE}/${CONTAINER_NAME}"
        FULL_IMAGE = ""
    }
    stages {
        stage('get image tags') {
            steps {
                script {
                    // 获取可用 tag 列表
                    def tags = getTagsFromAPI(env.SERVICE_NAME)
                    // 使用 input 步骤让用户选择 tag
                    def selectedTag = input(
                        message: '请选择要部署的镜像 Tag',
                        parameters: [
                            choice(
                                name: 'IMAGE_TAG',
                                choices: tags,
                                description: '可用镜像 Tag 列表'
                            )
                        ]
                    )
                    // 将选择的 tag 保存到环境变量
                    env.IMAGE_TAG = selectedTag
                    // 拼接全局镜像变量
                    env.FULL_IMAGE = "${env.DOCKER_OUTSIDE_REGISTRY}/${env.DOCKER_NAMESPACE}/${env.SERVICE_NAME}:${env.IMAGE_TAG}"
                    echo "准备镜像回滚服务: ${env.SERVICE_NAME}"
                    echo "使用镜像: ${env.FULL_IMAGE}"
                }
            }
        }
        stage('publish cn on xxx'){
            steps {
                script {
                sshagent(credentials: ['']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no root@xxx /bin/bash -c '
                    '
                    """
                }
                }
            }
        }
        stage('publish cn on xxxx'){
            steps {
                sshagent(credentials: ['']) {
                sh """
                 ssh -o StrictHostKeyChecking=no root@xxxx /bin/bash -c '
                '
                """
                }
            }
        }
    }
}

// 从动态 URL 获取标签列表
def getTagsFromAPI(serviceName) {
    def url = "https://xxxx/image_record/${serviceName}"
    try {
        def response = sh(script: "curl -s ${url}", returnStdout: true).trim()
        // 检查响应是否为空
        if (!response) {
            echo "API 返回空响应，使用默认 tag"
            return ['latest']
        }
        // 解析 JSON 响应
        def json = readJSON text: response
        if (json.code == 404) {
            echo "API 返回 404 : ${json.msg ?: '无错误信息'}，使用默认 tag"
            return ['latest']
        }
        if (json.tag && json.tag instanceof List && !json.tag.isEmpty()) {
            return json.tag
        } else {
            echo "tag 字段不存在或为空，使用默认 tag"
            return ['latest']
        }
    } catch (Exception e) {
        echo "获取标签列表失败: ${e.getMessage()}"
        return ['latest'] // 返回默认选项
    }
}
```

