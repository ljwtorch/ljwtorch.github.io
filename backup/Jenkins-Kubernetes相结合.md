
## 1.安装/部署

### 1.1 docker compose

`docker-compose.yml`

```yaml
services:                                     
  jenkins:
    restart: always                            
    image: jenkins/jenkins:2.414.3  
    #image: jenkins/jenkins:2.387.2 
    container_name: jenkins
    network_mode: host
    ports:                                     
      - 18080:8080                              
      - 28888:28888
    volumes:
      - ./:/var/jenkins_home  
      - /usr/bin/docker:/usr/bin/docker               
      - /usr/local/bin/docker-compose:/usr/local/bin/docker-compose
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Asia/Shanghai
      - JENKINS_SLAVE_AGENT_PORT=28888
```

## 2.Pipeline(流水线任务)

### 2.1 Jenkinsfile概览

*测试环境将服务部署到k8s中的Jenkinsfile，包括了项目代码拉取、配置文件拉取、项目编译、docker镜像构建/上传、多agent-workspace操作、Pod首次发布和滚动更新检测*

```groovy
pipeline {
    agent {
        label 'node_X'
    }
    tools {
        jdk 'jdk11'
    }
    environment {
        //一般只需要修改这里几个
        CONTAINER_NAME = 'xxx'	//容器tag
        YAML_NAME = 'xxx-deploy.yml'	//yml文件名
        PUB_ENV = 'test'	//发布环境，对应以后进行版本控制的目录位置
        K8S_NAMESPACE = 'test-xxx'
        PROJECT_GIT_URL = ''       //项目代码地址
        CONFIG_GIT_URL = ''        //配置文件地址(当项目Jenkinsfile和项目不在同一仓库，这里指存放所有Jenkinsfile的仓库)

        DOCKER_REGISTER_CREDS = credentials('registry_xxx')	//docker registry凭证
        DOCKER_REGISTRY = "xxxxx"	//Docker仓库地址
        DOCKER_NAMESPACE = "xxxxx"	//命名空间
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${CONTAINER_NAME}"	//组合成完整的镜像名称
    }
    stages {
        stage('get timestamp') {
            steps {
                script {
                    //获取当前时间戳
                    def timestamp = sh(returnStdout: true, script: 'date +%Y%m%d%H%M%S').trim()
                    //时间戳赋值
                    env.TIMESTAMP = timestamp
                }
            }
        }
        stage('fetch code') {
            steps {
                echo '----------Fetch code----------'
                git branch: 'release', credentialsId: 'git-credentials-backend', url: PROJECT_GIT_URL
            }
        }
        stage('fetch config') {
            agent {
                label 'node_xxx'	//在stage步骤中可以穿插在其他agent的操作步骤
            }
            steps {
                cleanWs() // 清理工作目录，防止工作目录产生其他产物
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
                """
            }
        }
        stage('image push'){
            steps {
                sh """
                    docker login -u ${DOCKER_REGISTER_CREDS_USR} -p=${DOCKER_REGISTER_CREDS_PSW} ${DOCKER_REGISTRY}
                    docker push ${DOCKER_IMAGE}:${env.TIMESTAMP}
                    docker rmi ${DOCKER_IMAGE}:${env.TIMESTAMP}
                """
            }
        }
        stage('publish cn on k8s'){
            agent {
                label 'node_XXX'
            }
            steps {
                script {
                    // 运行kubectl命令，检查旧版本是否存在于集群中
                    def deploymentsName = "bash -c 'kubectl get deployments -n ${K8S_NAMESPACE} ${CONTAINER_NAME} --output=jsonpath={.metadata.name}; exit 0;'"
                    def result = sh(returnStdout: true, script: deploymentsName).trim()
                    echo "即将执行的服务: ${result}"

                    if (result=="${CONTAINER_NAME}") {
                        // 旧版本已存在于集群中，执行kubectl set image更新镜像
                        echo "执行镜像滚动更新！"
                        sh """
                            kubectl set image -n ${K8S_NAMESPACE} deployment/${CONTAINER_NAME} ${CONTAINER_NAME}=${DOCKER_IMAGE}:${env.TIMESTAMP} --record
                        """
                    } else {
                        // 旧版本不存在于集群中，先执行kubectl apply部署
                        echo "服务不存在集群中，执行首次部署!"
                        sh """
                            kubectl apply -f ../jenkinsfile/${PUB_ENV}/${K8S_NAMESPACE}/${YAML_NAME}
                            kubectl set image -n ${K8S_NAMESPACE} deployment/${CONTAINER_NAME} ${CONTAINER_NAME}=${DOCKER_IMAGE}:${env.TIMESTAMP} --record
                        """
                    }
                }
            }
        }
        stage('Check Pod Status') {
            agent {
                label 'node_xxx'
            }
            steps {
                script {
                    def namespace = env.K8S_NAMESPACE
                    def containerName = env.CONTAINER_NAME
                    def timeoutSeconds = 120

                    //等待滚动更新完成
                    sh "kubectl rollout status deployment -n ${namespace} ${CONTAINER_NAME} --timeout=${timeoutSeconds}s"
                    sh "sleep 5s"
                    // 获取Pod名称
                    def podName = sh(
                        script: "kubectl get pods -l app=${containerName} -n ${namespace} -o jsonpath='{.items[0].metadata.name}'",
                        returnStdout: true
                    ).trim()

                    // 检查容器状态
                    def containerReady = sh(
                        script: "kubectl get pods ${podName} -n ${namespace} -o jsonpath='{.status.containerStatuses[0].ready}'",
                        returnStdout: true
                    ).trim()

                    if (containerReady == 'true') {
                        echo "Container ${containerName} in Pod ${podName} is ready. ContainerIsReady:${containerReady}"
                    } else {
                        error "Container ${containerName} in Pod ${podName} is not ready. ContainerIsReady:${containerReady}"
                    }
                }
            }
        }
    }
}

```

### 2.2 镜像发布到K8S

*见 2.1中 `publish cn on k8s`步骤*

### 2.3 滚动更新/更新后检测

*见 2.1 中 `Check Pod Status` 步骤*