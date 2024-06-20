## 1.部署

### 1.1 Solo

独立服务器是Azkaban的独立实例，也是最简单的开始。独立服务器具有以下优点

- **易于安装** - 不需要MySQL实例。它使用H2作为其主要的持久性存储。
- **易于启动** - Web服务器和执行器服务器都在相同的过程中运行
- **全功能** - 它包装所有的Azkaban功能。您可以正常使用它并为其安装插件

按照以下步骤开始。

1. **克隆仓库：**运行`git clone https://github.com/azkaban/azkaban.git`
2. **构建Azkaban并创建安装**运行`cd azkaban; ./gradlew build installDist`
3. **启动服务器：**运行`cd azkaban-solo-server/build/install/azkaban-solo-server; bin/azkaban-solo-start.sh`
4. **停止服务器：**运行`bin/azkaban-solo-shutdown.sh`从azkaban-solo-server安装目录中

编译安装报错解决办法：

1. 编译安装指令：`./gradlew clean build -x test` -x test是为了跳过测试阶段；

2. 配置编译镜像源：

   ```shell
   vim build.gradle
     repositories {
           maven {
               url 'https://maven.aliyun.com/repository/public/'
           }
           mavenCentral()
           maven {
               url 'https://maven.aliyun.com/repository/gradle-plugin/'
           }
     }
   ```

3. 安装node，npm不使用默认的Download：

   ```shell
   # true改为false
   node {
       // Version of node to use.
       version = '8.10.0'
   
       // Version of npm to use.
       npmVersion = '5.6.0'
   
       // Base URL for fetching node distributions (change if you have a mirror).
       distBaseUrl = 'https://nodejs.org/dist'
   
       // If true, it will download node using above parameters.
       // If false, it will try to use globally installed node.
       download = false
   
       // Set the work directory for unpacking node
       workDir = file("${project.buildDir}/nodejs")
   
       // Set the work directory where node_modules should be located
       nodeModulesDir = file("${project.projectDir}")
   }
   ```

4. 编译完成后拷贝出各个模块build/distributions下的压缩包即可；

### 1.2 集群

1. 安装MySQL；

   ```shell
   docker run -d -p 3306:3306 --name mysql \
   -v ./logs:/var/log/mysql \
   -v ./data:/var/lib/mysql \
   -v ./conf:/etc/mysql \
   --restart=always \
   -e MYSQL_ROOT_PASSWORD=root \
   -d mysql:5.7.31
   
   #skip-grant-tables
   ```

2. 设置数据库：

   ```shell
   # 为Azkaban创建一个数据库。例如：
   mysql> CREATE DATABASE azkaban;
   # 为Azkaban创建一个数据库用户。例如：
   mysql> CREATE USER 'username'@'%' IDENTIFIED BY 'password';
   # 设置数据库的用户权限。 为Azkaban创建一个用户（如果尚未创建），并为Azkaban数据库中的所有表赋予用户INSERT，SELECT，UPDATE，DELETE权限。
   mysql> CREATE USER 'username'@'%' IDENTIFIED BY 'password';
   # 配置数据包大小可能需要配置。默认情况下，MySQL可能有一个可接受的低数据包大小。为了增加它，你需要将属性max_allowed_packet设置为更高的值，比如1024M。
   ```

3. 创建Azkaban表：

   ```shell
   CREATE DATABASE azkaban;
   CREATE USER 'azkaban'@'%' IDENTIFIED BY 'azkaban';
   GRANT SELECT,INSERT,UPDATE,DELETE ON azkaban.* to 'azkaban'@'%' WITH GRANT OPTION;
   # 加载mysql表
   mysql> source /opt/model/azkaban/azkaban-db-3.84.4/create-all-sql-3.84.4.sql
   ```

4. 启动azkaban-exec

   ```shell
   # 要进入到exec的根目录中执行bash脚本，否则会因为配置文件中的相对路径而报错
   bash bin/start-exec.sh
   curl -G "node0:12321/executor?action=activate" && echo
   # 运行的每台机器都要执行，每次服务关闭重新打开后都需要激活，注意数据库中的execute表中激活值为1，未激活为0
   ```

5. 启动azkaban-web

   ```shell
   # 用户配置文件
   vim azkaban-users.xml
   # 添加一行用户配置
   <user password="root" roles="admin" username="root"/>
   
   ```

# 进入到web的根目录执行

`bash bin/start-web.sh`

1. 同一文件夹下创建azkaban.project

   ```yaml
   azkaban-flow-version: 2.0
   # 该文件作用是采用新的flow-api的方式解析flow文件
   ```

2. 新建basic.flow文件

   ```yaml
   nodes:
     - name: jobA	# Job名称
       type: command 	# Job类型
       config:	# Job配置
         command: echo "Hello World"
   ```

3. 将存放上述两个文件的文件夹压缩到一个zip文件中，文件必须为英文，在web端创建任务并上传；

### 2.2 任务作业依赖案例

> JobA和JobB执行结束后才能执行JobC

1. basic.flow

   ```yaml
   nodes:
     - name: jobC
       type: command
       # jobC 依赖 JobA和JobB
       dependsOn:
         - jobA
         - jobB
       config:
         command: echo "I’m JobC"
   
     - name: jobA
       type: command
       config:
         command: echo "I’m JobA"
   
     - name: jobB
       type: command
       config:
         command: echo "I’m JobB"
   ```

2. 压缩、运行任务步骤同上

### 2.3 自动失败重试案例

1. basic.flow

   ```yaml
   nodes:
     - name: JobA
       type: command
       config:
         command: sh /not_exists.sh
         retries: 3	# 重试次数
         retry.backoff: 10000	# 重试时间间隔
   ```

   也可以在Flow全局配置中添加任务失败重试配置，此位置会应用到所有Job：

   ```yaml
   config:
     retries: 3
     retry.backoff: 10000
   nodes:
     - name: JobA
       type: command
       config:
         command: sh /not_exists.sh
   ```

2. 压缩、运行任务步骤同上；

### 2.4 手动失败重试案例

1. basic.flow

   ```yaml
   nodes:
     - name: JobA
       type: command
       config:
         command: echo "This is JobA."
   
     - name: JobB
       type: command
       dependsOn:
         - JobA
       config:
         command: echo "This is JobB."
   
     - name: JobC
       type: command
       dependsOn:
         - JobB
       config:
         command: echo "This is JobC."
   
     - name: JobD
       type: command
       dependsOn:
         - JobC
       config:
         command: echo "This is JobD."
   
     - name: JobE
       type: command
       dependsOn:
         - JobD
       config:
         command: echo "This is JobE."
   
     - name: JobF
       type: command
       dependsOn:
         - JobE
       config:
         command: echo "This is JobF."
   ```

2. 压缩、运行任务步骤同上；

3. Enable和Disable下面都分别有如下参数：

   1. Parents：该作业的上一个任务
   2. Ancestors：该作业前的所有任务
   3. Children：该作业后的一个任务
   4. Descendents：该作业后的所有任务
   5. Enable All：所有的任务