 on AWS实现基于jenkins pipeline的CI/CD

## 目的

​    本项目主要目的为在k8s on AWS上实现基于jenkins pipeline的CI/CD ，其中会利用到Github Private为托管代码仓库，ECR（AWS 托管镜像仓库）为私有镜像仓库，k8s pod作为jenkins slave，jenkins pipeline作为流程引擎

##项目架构

![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/whole_pic.png)



1. 推送代码到托管镜像仓库
2. Github 基于webhook触发jenkins pipeline项目
3. Jenkins master分配kubernetes slave作为项目的执行环境，同时k8s启动slave pod
4. Jenkins slave pod运行pipeline中指定的任务第一步从私有代码仓库拉下代码
5. Jenkins slave pod执行代码测试，测试完毕后依据代码仓库格式，构造镜像
6. jenkins slave pod推送镜像到ECR上
7. jenkins slave pod执行应用服务的更新任务
8. 应用服务pod所在节点拉取相应的镜像，完成镜像的替换，即应用的更新

##	前提条件

1. 已部署k8s集群 [部署连接](https://github.com/nwcdlabs/kops-cn/blob/master/README_en.md#HOWTO) ，并且设置了国内docker源
2. Clone 此repo到本地

## 步骤一:  安装jenkins server于k8s之上（直接部署）

如果**不关心配置的细节**，可以执行此步骤，然后跳转到**步骤二的第7步**

```
$ cd container/asset
```
**如果是北京区**, 请执行
```
$ sed -i 's/cn-northwest-1/cn-north-1/g' sc.yaml
```
一键部署
```
$ kubectl apply -f .
```
## 步骤二:  安装jenkins server于k8s之上 （细节介绍）

**如果已经执行步骤一，可直接跳过执行步骤三**

1. 配置Jenkins server所需要的存储盘

   进入项目目录
   ```
   $ cd container/asset
   ```
   查看对应的存储配置文件
   ```
   $ cat sc.yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: ebs-gp2
   provisioner: kubernetes.io/aws-ebs
   parameters:
     type: gp2 
     zones: cn-northwest-1a #限定生成的区域
     fsType: ext4  #默认的存储格式
   reclaimPolicy: Retain  # 当删除pvc的时候，对应的存储卷不回收
   allowVolumeExpansion: true # 使得pvc可以在线扩容
   ```
   通过上述文件，我们可以配置了一个可用的StorageClass，其中需要注意的是，如果 **需要部署在北京区**，请执行以下操作，如果**部署在宁夏区保持默认**即可
   ```
   sed -i 's/cn-northwest-1/cn-north-1/g' sc.yaml
   sed -i 's/cn-northwest-1/cn-north-1/g' jenkins.yaml
   ```
   
2. 配置相应的存储大小请求

   配置好相应的存储类后，我们就可以声明jenkins所需要的存储大小，执行以下操作
   ```
   $ cat jenskin-volum.yaml
   kind: PersistentVolumeClaim
   apiVersion: v1
   metadata:
     name: jenkins-pvc
     namespace: kube-jenkins  # 存储声明的namespace
   spec:
     storageClassName: ebs-gp2 #上一步声明的存储类的名字
     accessModes:
       - ReadWriteOnce # 同时只有一个pod能够mount，ebs唯一支持的格式
     resources:
       requests:
         storage: 20Gi # 声明的存储大小
   ```
   其中具体的参数，可以参照注释缩写
   
3. 配置jenkins server所需要的k8s权限

   配置jenkins server在kube-jenkinsnamespace具有足够的权限

   ```
   $ cat jenkins-rbac.yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: jenkins-server
     namespace: kube-jenkins
   
   ---
   
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1beta1
   metadata:
     name: jenkins-server
   rules:
     - apiGroups: ["extensions", "apps"]
       resources: ["deployments"]
       verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
     - apiGroups: [""]
       resources: ["services"]
       verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
     - apiGroups: [""]
       resources: ["pods"]
       verbs: ["create","delete","get","list","patch","update","watch"]
     - apiGroups: [""]
       resources: ["pods/exec"]
       verbs: ["create","delete","get","list","patch","update","watch"]
     - apiGroups: [""]
       resources: ["pods/log"]
       verbs: ["get","list","watch"]
     - apiGroups: [""]
       resources: ["secrets"]
       verbs: ["get"]
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRoleBinding
   metadata:
     name: jenkins-server
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: jenkins-server
   subjects:
     - kind: ServiceAccount
       name: jenkins-server
       namespace: kube-jenkins
   ```

4. 配置jenkins server的部署文件

   ```
   $ cat jenkins.yaml
   apiVersion: extensions/v1beta1
   kind: Deployment
   metadata:
     name: jenkins-server
     namespace: kube-jenkins
   spec:
     template:
       metadata:
         labels:
           app: jenkins-server
       spec:
         terminationGracePeriodSeconds: 10
         serviceAccountName: jenkins-server
         containers:
         - name: jenkin-server
           image:  jenkins/jenkins:lts
           imagePullPolicy: IfNotPresent
           
           ....
           
           volumeMounts:
           - name: jenkinshome
             subPath: jenkins-server
             mountPath: /var/jenkins_home
         nodeSelector:
           failure-domain.beta.kubernetes.io/zone: failure-domain.beta.kubernetes.io/zone=cn-northwest-1a
         securityContext:
           fsGroup: 1000
         volumes:
         - name: jenkinshome
           persistentVolumeClaim:
             claimName: jenkins-pvc
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: jenkins-server
     namespace: kube-jenkins
     labels:
       app: jenkins-server
   spec:
     selector:
       app: jenkins-server
     type: LoadBalancer
     ports:
     - name: web
       port: 8080
       targetPort: web
     - name: agent
       port: 50000
       targetPort: agent
   ```

   其中上面上面配置文件中的下述部分

   ```
    nodeSelector:
           failure-domain.beta.kubernetes.io/zone: failure-domain.beta.kubernetes.io/zone=cn-northwest-1a
   ```

   因为**EBS卷并不能跨可用区**，所以我们需要**限定对应的PV(EBS卷)在同一可用区**，这样当jenkins master出现故障的时候，可以实现可用区内jenkins master的自动恢复。

   > 如果要实现region级别的高可用，可以参考[CSI-S3](https://github.com/ctrox/csi-s3) 作为StorageClass 
   >
   > 如果要实现jenkins数据的自动快照，可参考[Snapshot](https://github.com/lab798/aws-auto-snapshot)

5. 配置jenkins server 和 job运行的namespace

   ```
   $ cat jenkins-ns.yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: kube-jenkins
   ```

6. 部署

   ```
   $ kubectl apply -f .
   ```

7. 查看部署状态

   ```
   $ kubectl get pods -w -nkube-jenkins
   ```

   当出现如下所示结果，表明pod启动正常 

   ``` 
   $ kubectl get pods -w -nkube-jenkins
   NAME                              READY   STATUS    RESTARTS   AGE
   jenkins-server-8658b744fc-9mnmq   1/1     Running   8          7h27m
   ```

   此时通过读取应用日志获取jenkins的初始化密码

   ```
   $ kubectl logs jenkins-server-8658b744fc-9mnmq -nkube-jenkins
   ```

   找到如下语句

   ```
   Jenkins initial setup is required. An admin user has been created and a password generated.
   Please use the following password to proceed to installation:
   
   f1820659b5bd44**091c132819932c6ac  #此处为初始化密码
   
   This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
   ```

   其中加注释处即为初始化密码，建议记录在记事本中

8. 查看访问地址

   ```
   $ kubectl get svc -nkube-jenkins
   NAME             TYPE           CLUSTER-IP   EXTERNAL-IP                                                                       PORT(S)                          AGE
   jenkins-server   LoadBalancer   10.0.2.147   a6c85f1b2cda011e9b41502c1675409f-1696512250.cn-northwest-1.elb.amazonaws.com.cn   8080:32555/TCP,50000:32443/TCP   7h55m
   ```
   其中**a6c85f1b2cda011e9b41502c1675*t-1.elb.amazonaws.com.cn**对应的即为对外暴露的负载均衡器地址

## 步骤二 :  配置jenkins为利用k8s pod启动 jenkins slave 

1. jenkins 初始化

   根据上文中的负载均衡器地址，在浏览器中访问页面**负载均衡地址:8080**, 进入界面后，在其中输入在步骤一.7中得到的初始密码，并点击next；进入后点击 **install suggested plugin**，进入下图界面：

   ![jenkins 初始化](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.1+jenkins+init.png)

   安装完毕后会指引用户创建第一个admin 用户，输入相关用户名和密码点击保存和继续，从而进入jenkins主界面

2. kubernetes插件安装

   如下图红框所示，点击Manage Jenkins -> Mange Plugins

   ![kubernetes插件](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.2.1+kubernetes+plugin+enter.png)

   在插件页面，如下图所示搜索kubernetes，并选择对应选项，然后点击install

   ![安装jenkins](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.2+kubernetes+plugin.png)

   等待安装完毕后，浏览器输入**jenkins url:8080/restart**，重启jenkins

3. 配置jenkins于kubernetes集成

   安装好kubernetes组件后，接下来我们需要配置jenkins slave作为kubenetes的pod进行执行。如下图所示，进入jenkins的全局配置界面

   ![进入配置](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.3+main+page.png) 

   进入configure system后拉到页面最下方，如图所示，**点击add a new cloud**，然后选择kubernetes

   ![add cloud](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.3.2+add+cloud.png)

   如下图进行配置，

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.3.3+new+cloud.png)

   配置完毕后，点击**右下方**的**test connection**，如果出现 Connection test successful 的提示信息证� Jenkins 已经可以和 Kubernetes 系统正常通信了；如果测试不成功，请仔细检查参数是否正确

   > 其中server名可以通过 kubectl get svc -n somens 得到默认somens namespace下的所有service

   

4. 配置jenkins slave pod参数

   配置jenkins-slave pod如下图所示, 其中由于插件自身的问题，Name一定要写为[jnlp](https://stackoverflow.com/questions/42124958/connection-dropped-for-jenkins-slave-on-kubernetes)

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.4.1+pod+config.png)

​       其中需要注意的是上图的**Labels**以及其下一栏的 **only build jobs ...**, 两者组合起来表示，后续再执行jenkins任务的时候，只有指定**node labels**为 **jenkins-slave-k8s**才会利用kubernetes pod 的模式执行jenkins job

> 如果在之后运行job的过程中出现镜像一直无法pull下来的状况，可以更改上面的docker image为 182335798701.dkr.ecr.cn-northwest-1.amazonaws.com.cn/jenkins-slave:jnlp6

​       接下来，因为jenkins slave执行在docker之中，而slave之中又要执行docker，kubectl等操作，所以我们需要把宿主机（对应的worker）上的环境mount到docker之中，配置如下

​		![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.4.2+volume+mount.png)

> 挂载docker.sock是为了和docker.daemon进行通信
>
> 挂载.kube是为了获取执行kubectl指令的配置参数

​      此时点击左下角的save，整个jenkins与k8s的集成便完成了  

5. 简单测试

   进入jenkins主界面，点击new-item, 进入如下界面，创建名为jenkins-demo的**Freestyle project**

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.5.1+freestyle.png)

    配置选择kubernetes pod为jenkins job的执行节点，**此处的label一定要与步骤二.4中写的label进行匹配**

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.5.2+label.png)

   往下滑动到build step，选择下图中的**Execut Shell**选项

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.5.3+shell.png)

   往下滑动到build step，选择下图中的**Execut Shell**选项

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.5.4+shell+script.png)

   其中

   > docker info 是为了查看能否成功调用docker daemon
   > kubectl get pod 是为了查看是否调用宿主机的.kube配置文件，从而连接apiserver

​      点击保存后，此时处于项目的界面，点击右侧的build now，开始构建

   > 如果构建过程中一直出现，worker offline等job pending的状况，可以从jenkins -> Manage Jenkins -> System log处查看是否有错误提示

​      构建成功后，build hisotry处会出现如下的标识（蓝色）

​     ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/2.5.5+build+success.png)

6. 点击上图中的2，或者任意数字，然后点击console output 得到类似如下的输出

   ``` kubectl  get pods 
   + echo #######docker cli ###
   #######docker cli ###
   + docker info
   Containers: 9
   *******
   + echo #######kubectl cli #####
   #######kubectl cli #####
   + kubectl get pod
   NAME                              READY   STATUS    RESTARTS   AGE
   jenkins-server-8658b744fc-4r47f   1/1     Running   0          161m
   jnlp-3qjfw                        1/1     Running   0          5s
   Finished: SUCCESS
   
   ```

   这样说明我们整个流程已经配通

## 步骤三 :  配置主动推送模式下的Pipeline

### 实验说明

  本实验是基于github private repo进行的，所以请先在github上创建好私有仓库

1. 配置代码仓库

   下载模板代码库

   ```
   $ git clone https://github.com/jansony1/jenkins-new-public.git
   ```

   配置源为实验者自身代码仓库

   ```
   $ git remote add origin your_private_repo_url
   $ git push -u origin master
   ```

2. 安装插件

   因为本实验要用到AWS ECR这样的托管镜像仓库，对于托管镜像仓库需要利用aws的AKSK进行签名发送请求，才能够进行正常 ` docker pull ` 等相关操作。所以首先我们要进入**Manage Jenkins -> Mange Plugins**，安装如下插件
   **IAM 插件**
   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.1+install+plugin.png)
   ​ **ECR 插件**![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.1.1+ECR.png)
   ​   安装完毕后，web界面执行Jenkins-url:8080/restart操作

3. 秘钥配置

   本实验中，访问私有的Github Repo以及AWS ECR都需要配置相关的秘钥，首先我们先配置两个秘钥

   **Github 秘钥配置** 

   在jenkins的主界面，进入Credentials->System界面，点击Global Credentials，出现如下界面

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.3+configure+user.png)

   点击上图红框的 Add Credentials，如下图所示进行配置

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.2.1+github+credentials.png)
   其中

   * Username为github登录的用户名
   * Password为github对应的密码
   * Description可以填写为有意义的描述，以便日后区分

   点击确认，github访问的秘钥即创建完毕

   **ECR 秘钥配置**

   对于ECR相关的访问秘钥，实际上就是在配置对应的AKSK

4. Pipeline项目逻辑

   对于Jenkins Pipeline的逻辑，首先可以参考下图

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/jenkins+logic.png)
   其中：

    - **Jenkins Master**：负责提供整个集群的管理，任务的调度等工作
    - **Node**：在jenkins中具体执行任务的为Jenkins slave，而node的概念即是指定slave的工作环境，比如本实验，我们设定了kubernetes pod作为工作环境；那么当我们在设定jenkins 具体任务时，指定node的label为:jenkins-slave-k8s，即可利用kubernetes pod运行具体的任务
    - **Stage**: 每个Jenkins Slave执行的任务可以分为多个顺序的Stage，比如先代码下载Stage，编译Stage，测试Stage,部署Stage等等

5. Pipeline项目配置

   进入下图界面，创建新的Pipeline项目

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.4.1+project+configure.png)

   创建完毕后拉到最下方

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.4.2+pipeline+configure.png)

   其中Script 处就是我们写Pipeline Script (Groovy)调度脚本的地方，下方列出了一个基本的模板

   ```
   node('jenkins-slave-k8s') {
       stage('Clone') {
           echo "1.Clone Stage"
           git credentialsId: 'c8d7ea58-aa4b-425b-b74e-51a066ab560b', url: 'https://github.com/jansony1/jenkins-new.git'
           script {
               build_tag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
               repo_name = '182335798701.dkr.ecr.cn-northwest-1.amazonaws.com.cn'
               app_name = 'jenkins-demo'
           }
       }
       stage('Test') {
         echo "2.Test Stage"
       }
       stage('Build') {
           echo "3.Build Docker Image Stage"
           sh "docker build -t ${repo_name}/${app_name}:${build_tag} ."
       }
       stage('Push') {
           echo "4.Push Docker Image Stage"
           withDockerRegistry(credentialsId:'ecr:cn-northwest-1:93afbbbf-4961-4206-8b7c-82db9dd4a55a', url: 'https://182335798701.dkr.ecr.cn-northwest-1.amazonaws.com.cn') {
              sh "docker push ${repo_name}/${app_name}:${build_tag}"
           }
       }
       stage('Deploy') {
           echo "5. Deploy Stage"
           def userInput = input(
               id: 'userInput',
               message: 'Choose a deploy environment',
               parameters: [
                   [
                       $class: 'ChoiceParameterDefinition',
                       choices: "Dev\nQA",
                       name: 'Env'
                   ]
               ]
           )
           echo "This is a deploy step to ${userInput}"
           sh "sed -i 's/<REPO_NAME>/${repo_name}/' echo-server.yaml"
           sh "sed -i 's/<APP_NAME>/${app_name}/' echo-server.yaml"
           sh "sed -i 's/<BUILD_TAG>/${build_tag}/' echo-server.yaml"
           if (userInput == "Dev") {
               // deploy dev stuff
           } else if (userInput == "QA"){
               // deploy qa stuff
           } else {
               // deploy prod stuff
           }
           sh "kubectl apply -f echo-server.yaml"
       }
   }
   ```

   在上面的代码中，我们首先指定了pipeline的运行环境为**jenkins-slave-k8s**, 然后该job中包含了一个基本的CICD所必备的五个环节：**Git Clone ->Test->Build->Push->Deploy**
   其中，有几个部分需要用户修改为自己环境的相关参数

   * **Git Clone Stage**

   在该环节中，需要修改的为如下所示的Git相关的参数
       
   ```
   stage('1.Clone') {
           		**
               git credentialsId: '15ae7db7-1aa3-4dc9-88ba-b08b83a68646', url: 'https://github.com/jansony1/jenkins-new.git'
               **
    }
   ```
   此时我们可以借助如下图红框所示的脚本生成器进行相关的配置
   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.4.3+syntax.png)
   进入界面后，配置如下：
   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.4.4+syntax_git.png)
   ​其中上图中：

   * Repository URL:  之前创建的私有仓库地址
   * Branch: 指定对应的branch
   * Credentails: 选择在**步骤3.2**中配置的github相关秘钥 

   配置完毕后，点击上图中的Generate Pipeline Script，得到类似如下的输出
   ```
   git credentialsId: 'c8d7ea58-aa4b-425b-b74e-51a066ab560b', url: 'https://github.com/jansony1/jenkins-new.git'
   ```

   使用该配置，替换样本项目中的git配置; 同时，用户也需要用自己的ECR地址，修改下面的地址

   ```
    repo_name = '182335798701.dkr.ecr.cn-northwest-1.amazonaws.com.cn'
    app_name = 'jenkins-demo'
   ```

   * **Push Stage**

   此环节请替换对应DockerRegistry的配置
   ```
   stage('4.Push') {
           echo "4.Push Docker-Image Stage"
           withDockerRegistry(credentialsId: 'ecr:cn-northwest-1:44d5ee4f-232d-408d-bbf0-8ebbc8156fd0', url: 'https://182335798701.dkr.ecr.cn-northwest-1.amazonaws.com.cn') {
              sh "docker push ${repo_name}:${build_tag}"
           }
       }
   ```

   如Git Clone环节同理，配置ECR为Docker Registry为如下图所示

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.4.4+ecr.png)

   需要注意的是，Registry Credentials为在**3.2中配置的对应region的秘钥**，点击Generate Pipeline Script得到对应的语法

   ```
   withDockerRegistry(credentialsId:'ecr:cn-northwest-1:93afbbbf-4961-4206-8b7c-82db9dd4a55a', url: '182335798701.dkr.ecr.cn-northwest-1.amazonaws.com.cn') {
           // code here   
   }
   ```

   > 此时**需要注意**, 该插件存在一定的bug，无法为docker repo添加其协议头，即我们需要手动在上面的地址前面添加**https://**

   然后可以将生成的语法，替换掉模板处相关的信息

   **最后点击保存**，从而保存项目 

6. Pipeline项目测试

   点击**Build Now**后，我们会发现如下图所示，服务已经开始构建

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.5+build+now.png)

   如上图所示，我们能够在jenkins项目界面看到其所有的构造历史，每次的耗时，每次是否成功等等。如果要查看构建的详细输出，我们可以点击上图的**红圈的Build任务* 

   > 因为本项目由用户交互的存在，即输出部署的环境，所以必须点进具体的项目中，进行环境的的选择

   进入具体项目后，点击下图所示的Console log后，可以看到具体每一步的输出​   
   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.5+console+log.png)
   ​进入后，点击下面的input requested，选择需要部署的环境
   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/3.5.2+input+choice.png)
   ​最后界面看到如下的输出，表明CICD流程走通 

   ```
    + kubectl apply -f echo-server.yaml
    service/echo-demo unchanged
    deployment.apps/echo-demo created
    [Pipeline] }
    [Pipeline] // stage
    [Pipeline] }
    [Pipeline] // node
    [Pipeline] End of Pipeline
    Finished: SUCCESS
   ```
## 步骤四 :  基于Github Webhook的触发模式

1. 新建pipeline项目

   如步骤三所示，我们同样新建一个jenkins Pipeline的项目，打开项目后，首先选择下面的Build Triggers

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/4.1.1+pipeline+trigger.png)

   其含义为，当接收到Github的Webhook后，即触发本项目的构建。下一步我们配置接收到对应Webhook后所需要执行的工作源

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/4.1.2+pipeline+configure.png)

   其中

   * Repository URL:  相关github项目地址
   * Credentials：在步骤3.2中配置的github访问秘钥
   * Script Path: 在进入到相关github项目后，实际执行脚本的名字

   相关的Jenkinsfile在https://github.com/jansony1/jenkins-new-public已经提供, **其和我们在步骤三中使用的脚本一样，需要用户替换成自己的相关参数**, 此时我们可以从上一个项目中把Pipeline script粘贴下来，替换下面的文件内容

   ```
   $ vim Jenkinsfile
   ```

   替换完毕后，提交修改

   ```
   $ git add Jenkinsfile
   $ git commit -m 'modify Jenkinsfile'
   $ git push
   ```

2. 配置Github Webhook

   首先进入之前创建的github 私有仓库中，点击下图所示的settings

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/4.2.1+config.png)

   进入后点依次点击下图红框中的 **Webhook->add Webhook**

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/4.2.2+webhook.png)

   然后填写下图所示的**payload url**

   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/4.2.3+add+webhook.png)

   该url地址的格式为

   ```
   jenkins-url:8080/github-webhook/
   ```

   另外本实验中只配置了基于Push Event的触发，如果有基于Merge等请求的触发，可以点击上图中触发Events里面的 

   **Let me select individual event**

3. 测试

   修改本地README文件

   ```
   $ sed -i '$a add some new content'  README.md
   ```

   提交修改

   ```
   $ git add README.md
   $ git commit -m 'modify README'
   $ git push
   ```
   查看项目的Dashboard
   ![](https://zhenyu-github.s3-us-west-2.amazonaws.com/quick-start/4.3.1+result.png)
​   可以看到项目被成功构建
   
   >某些情况下，会出现push后无法触发的结果，此时

4. 错误排除

   如果出现不能触发的情况，应该

   * 首先检查github处webhook的发送是否成功
   * 查看jenkins system log 看是否成功收到webhook通知
   * 某些情况下，需要先执行一遍步骤三的主动触发，才能够顺利实现步骤四的被动触发，[详情见](https://issues.jenkins-ci.org/browse/JENKINS-35132)

​	

## 下一章 :  

## 基于Github Webhook的multi-branch触发模式

##集成GITLAB private repo
















