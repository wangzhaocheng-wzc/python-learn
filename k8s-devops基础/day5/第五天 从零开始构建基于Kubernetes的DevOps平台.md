### 基于Kubernetes的DevOps平台实践

持续集成工具：

- Jenkins
- gitlabci
- Tekton 

本章基于k8s集群部署gitlab、sonarQube、Jenkins等工具，并把上述工具集成到Jenkins中，以Django项目和SpringBoot项目为例，通过多分支流水线及Jenkinsfile实现项目代码提交到不同的仓库分支，实现自动代码扫描、单元测试、docker容器构建、k8s服务的自动部署。

- DevOps、CI、CD介绍
- Jenkins、sonarQube、gitlab的快速部署
- Jenkins初体验
- 流水线入门及Jenkinsfile使用
- Jenkins与Kubernetes的集成
- sonarQube代码扫描与Jenkins的集成
- 实践Django项目的基于Jenkinsfile实现开发、测试环境的CI/CD

#### DevOps、CI、CD介绍

 Continuous Integration (*CI*) / Continuous Delivery (*CD*)  

软件交付流程

![](images\devops-roles.jpg)

一个软件从零开始到最终交付，大概包括以下几个阶段：规划、编码、构建、测试、发布、部署和维护，基于这些阶段，我们的软件交付模型大致经历了几个阶段：

##### 瀑布式流程

![](images\devops-waterfall.jpg)

前期需求确立之后，软件开发人员花费数周和数月编写代码，把所有需求一次性开发完，然后将代码交给QA（质量保障）团队进行测试，然后将最终的发布版交给运维团队去部署。瀑布模型，简单来说，就是等一个阶段所有工作完成之后，再进入下一个阶段。这种模式的问题也很明显，产品迭代周期长，灵活性差。一个周期动辄几周几个月，适应不了当下产品需要快速迭代的场景。

##### 敏捷开发

![](images\devops-agile.jpg)



任务由大拆小，开发、测试协同工作，注重开发敏捷，不重视交付敏捷

##### DevOps

![](images\devops-compire.jpg)

开发、测试、运维协同工作, 持续开发+持续交付。

我们是否可以认为DevOps = 提倡开发、测试、运维协同工作来实现持续开发、持续交付的一种软件交付模式？

大家想一下为什么最初的开发模式没有直接进入DevOps的时代？

原因是：沟通成本。

各角色人员去沟通协作的时候都是手动去做，交流靠嘴，靠人去指挥，很显然会出大问题。所以说不能认为DevOps就是一种交付模式，因为解决不了沟通协作成本，这种模式就不具备可落地性。

那DevOps时代如何解决角色之间的成本问题？DevOps的核心就是自动化。自动化的能力靠什么来支撑，工具和技术。

DevOps工具链

![](images\devops-tools.jpg)

靠这些工具和技术，才实现了自动化流程，进而解决了协作成本，使得devops具备了可落地性。因此我们可以大致给devops一个定义：

devops = 提倡开发、测试、运维协同工作来实现持续开发、持续交付的一种软件交付模式 + 基于工具和技术支撑的自动化流程的落地实践。

因此devops不是某一个具体的技术，而是一种思想+自动化能力，来使得构建、测试、发布软件能够更加地便捷、频繁和可靠的落地实践。本次课程核心内容就是要教会大家如何利用工具和技术来实现完整的DevOps平台的建设。我们主要使用的工具有：

1. gitlab，代码仓库，企业内部使用最多的代码版本管理工具。
2. Jenkins， 一个可扩展的持续集成引擎，用于自动化各种任务，包括构建、测试和部署软件。
3. robotFramework， 基于Python的自动化测试框架
4. sonarqube，代码质量管理平台
5. maven，java包构建管理工具
6. Kubernetes
7. Docker



#### Jenkins初体验

##### Kubernetes环境中部署jenkins

[其他部署方式](https://jenkins.io/zh/doc/book/installing/)

注意点：

1. 第一次启动很慢
2. 因为后面Jenkins会与kubernetes集群进行集成，会需要调用kubernetes集群的api，因此安装的时候创建了ServiceAccount并赋予了cluster-admin的权限
3. 默认部署到jenkins=true的节点
4. 初始化容器来设置权限
5. ingress来外部访问
6. 数据存储通过hostpath挂载到宿主机中

`jenkins/jenkins-all.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-master
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      devops: jenkins-master
  template:
    metadata:
      labels:
        devops: jenkins-master
    spec:
      nodeSelector:
        jenkins: "true"
      serviceAccount: jenkins #Pod 需要使用的服务账号
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /var/jenkins_home"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: jenkinshome
          mountPath: /var/jenkins_home
      containers:
      - name: jenkins
        image: jenkinsci/blueocean:1.23.2
        imagePullPolicy: IfNotPresent
        ports:
        - name: http #Jenkins Master Web 服务端口
          containerPort: 8080
        - name: slavelistener #Jenkins Master 供未来 Slave 连接的端口
          containerPort: 50000
        volumeMounts:
        - name: jenkinshome
          mountPath: /var/jenkins_home
        env:
        - name: JAVA_OPTS
          value: "-Xms4096m -Xmx5120m -Duser.timezone=Asia/Shanghai -Dhudson.model.DirectoryBrowserSupport.CSP="
      volumes:
      - name: jenkinshome
        hostPath:
          path: /var/jenkins_home/
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: slavelistener
    port: 50000
    targetPort: 50000
  type: ClusterIP
  selector:
    devops: jenkins-master
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-web
  namespace: jenkins
spec:
  rules:
  - host: jenkins.luffy.com
    http:
      paths:
      - backend:
          serviceName: jenkins
          servicePort: 8080
        path: /
```

创建服务：

```powershell
## 为k8s-slave1打标签，将jenkins-master部署在k8s-slave1节点
$ kubectl label node k8s-slave1 jenkins=true
## 部署服务
$ kubectl create -f jenkins-all.yaml
## 查看服务
$ kubectl -n jenkins get po
NAME                              READY   STATUS    RESTARTS   AGE
jenkins-master-767df9b574-lgdr5   1/1     Running   0          20s

# 查看日志，第一次启动提示需要完成初始化设置
$ kubectl -n jenkins logs -f jenkins-master-767df9b574-lgdr5
......
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

5396b4e1c395450f8360efd8ee641b18

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************

```

访问服务：

配置hosts解析，`172.21.51.67 jenkins.luffy.com`，然后使用浏览器域名访问服务。第一次访问需要大概几分钟的初始化时间。

![](images\jenkins_setup.jpg)

使用jenkins启动日志中的密码，或者执行下面的命令获取解锁的管理员密码：

```powershell
$ kubectl -n jenkins exec jenkins-master-767df9b574-lgdr5 bash 
/ # cat /var/jenkins_home/secrets/initialAdminPassword
35b083de1d25409eaef57255e0da481a

```

点击叉号，跳过选择安装推荐的插件环节，直接进入Jenkins。由于默认的插件地址安装非常慢，我们可以替换成国内清华的源，进入 jenkins 工作目录，目录下面有一个 `updates` 的目录，下面有一个 `default.json` 文件，我们执行下面的命令替换插件地址：

```powershell
$ cd /var/jenkins_home/updates
$ sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json 
$ sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json

```

> 暂时先不用重新启动pod，汉化后一起重启。

选择右上角admin->configure->password重新设置管理员密码，设置完后，会退出要求重新登录，使用admin/xxxxxx(新密码)，登录即可。

![](images\jenkins-mainpage.jpg)

##### 安装汉化插件

Jenkins -> manage Jenkins -> Plugin Manager -> Avaliable，搜索 `chinese`关键字

![](images\jenkins-install-plugins.jpg)

选中后，选择[Install without restart]，等待下载完成，然后点击[ Restart Jenkins when installation is complete and no jobs are running ]，让Jenkins自动重启



启动后，界面默认变成中文。



##### Jenkins基本使用演示

###### 演示目标

- 代码提交gitlab，自动触发Jenkins任务
- Jenkins任务完成后发送钉钉消息通知

###### 演示准备

*gitlab代码仓库搭建*

 https://github.com/sameersbn/docker-gitlab 



将网页中docker run 部署命令

```powershell
## 全量部署的组件
$ gitlab-ctl status
run: alertmanager: (pid 1987) 27s; run: log: (pid 1986) 27s
run: gitaly: (pid 1950) 28s; run: log: (pid 1949) 28s
run: gitlab-exporter: (pid 1985) 27s; run: log: (pid 1984) 27s
run: gitlab-workhorse: (pid 1956) 28s; run: log: (pid 1955) 28s
run: logrotate: (pid 1960) 28s; run: log: (pid 1959) 28s
run: nginx: (pid 2439) 1s; run: log: (pid 1990) 27s
run: node-exporter: (pid 1963) 28s; run: log: (pid 1962) 28s
run: postgres-exporter: (pid 1989) 27s; run: log: (pid 1988) 27s
run: postgresql: (pid 1945) 28s; run: log: (pid 1944) 28s
run: prometheus: (pid 1973) 28s; run: log: (pid 1972) 28s
run: puma: (pid 1968) 28s; run: log: (pid 1966) 28s
run: redis: (pid 1952) 28s; run: log: (pid 1951) 28s
run: redis-exporter: (pid 1971) 28s; run: log: (pid 1964) 28s
run: sidekiq: (pid 1969) 28s; run: log: (pid 1967) 28s
```

部署分析：

1. 依赖postgres
2. 依赖redis

使用k8s部署：

1. 准备secret文件

   ```powershell
   $ cat gitlab-secret.txt
   postgres.user.root=root
   postgres.pwd.root=
   
   $ kubectl -n jenkins create secret generic gitlab-secret --from-env-file=gitlab-secret.txt
   ```

   

2. 部署postgres

   注意点：

   - 使用secret来引用账户密码
   - 使用postgres=true来指定节点

   ```powershell
   $ cat postgres.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: postgres
     labels:
       app: postgres
     namespace: jenkins
   spec:
     ports:
     - name: server
       port: 5432
       targetPort: 5432
       protocol: TCP
     selector:
       app: postgres
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     namespace: jenkins
     name: postgres
     labels:
       app: postgres
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: postgres
     template:
       metadata:
         labels:
           app: postgres
       spec:
         nodeSelector:
           postgres: "true"
         tolerations:
         - operator: "Exists"
         containers:
         - name: postgres
           image:  172.21.51.67:5000/postgres:11.4 #若本地没有启动该仓库，换成postgres:11.4
           imagePullPolicy: "IfNotPresent"
           ports:
           - containerPort: 5432
           env:
           - name: POSTGRES_USER           #PostgreSQL 用户名
             valueFrom:
               secretKeyRef:
                 name: gitlab-secret
                 key: postgres.user.root
           - name: POSTGRES_PASSWORD       #PostgreSQL 密码
             valueFrom:
               secretKeyRef:
                 name: gitlab-secret
                 key: postgres.pwd.root
           resources:
             limits:
               cpu: 1000m
               memory: 2048Mi
             requests:
               cpu: 50m
               memory: 100Mi
           volumeMounts:
           - mountPath: /var/lib/postgresql/data
             name: postgredb
         volumes:
         - name: postgredb
           hostPath:
             path: /var/lib/postgres/
             
   
   #部署到k8s-slave2节点
   $ kubectl label node k8s-slave2 postgres=true
   
   #创建postgres
   $ kubectl create -f postgres.yaml
   
   # 创建数据库gitlab,为后面部署gitlab组件使用
   $ kubectl -n jenkins exec -ti postgres-7ff9b49f4c-nt8zh bash
   root@postgres-7ff9b49f4c-nt8zh:/# psql
   root=# create database gitlab;
   CREATE DATABASE
   ```

   

3. 部署redis

   ```powershell
   $ cat redis.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: redis
     labels:
       app: redis
     namespace: jenkins
   spec:
     ports:
     - name: server
       port: 6379
       targetPort: 6379
       protocol: TCP
     selector:
       app: redis
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     namespace: jenkins
     name: redis
     labels:
       app: redis
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: redis
     template:
       metadata:
         labels:
           app: redis
       spec:
         tolerations:
         - operator: "Exists"
         containers:
         - name: redis
           image:  sameersbn/redis:4.0.9-2
           imagePullPolicy: "IfNotPresent"
           ports:
           - containerPort: 6379
           resources:
             limits:
               cpu: 1000m
               memory: 2048Mi
             requests:
               cpu: 50m
               memory: 100Mi
               
   # 创建
   $ kubectl create -f redis.yaml
   ```

   

4. 部署gitlab

   注意点：

   - 使用ingress暴漏服务
   - 添加annotation，指定nginx端上传大小限制，否则推送代码时会默认被限制1m大小，相当于给nginx设置client_max_body_size的限制大小
   - 使用gitlab=true来选择节点
   - 使用服务发现地址来访问postgres和redis
   - 在secret中引用数据库账户和密码
   - 数据库名称为gitlab

   ```powershell
   $ cat gitlab.yaml
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: gitlab
     namespace: jenkins
     annotations:
       nginx.ingress.kubernetes.io/proxy-body-size: "50m"
   spec:
     rules:
     - host: gitlab.luffy.com
       http:
         paths:
         - backend:
             serviceName: gitlab
             servicePort: 80
           path: /
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: gitlab
     labels:
       app: gitlab
     namespace: jenkins
   spec:
     ports:
     - name: server
       port: 80
       targetPort: 80
       protocol: TCP
     selector:
       app: gitlab
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     namespace: jenkins
     name: gitlab
     labels:
       app: gitlab
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: gitlab
     template:
       metadata:
         labels:
           app: gitlab
       spec:
         nodeSelector:
           gitlab: "true"
         tolerations:
         - operator: "Exists"
         containers:
         - name: gitlab
           image:  sameersbn/gitlab:13.2.2
           imagePullPolicy: "IfNotPresent"
           env:
           - name: GITLAB_HOST
             value: "gitlab.luffy.com"
           - name: GITLAB_PORT
             value: "80"
           - name: GITLAB_SECRETS_DB_KEY_BASE
             value: "long-and-random-alpha-numeric-string"
           - name: GITLAB_SECRETS_DB_KEY_BASE
             value: "long-and-random-alpha-numeric-string"
           - name: GITLAB_SECRETS_SECRET_KEY_BASE
             value: "long-and-random-alpha-numeric-string"
           - name: GITLAB_SECRETS_OTP_KEY_BASE
             value: "long-and-random-alpha-numeric-string"
           - name: DB_HOST
             value: "postgres"
           - name: DB_NAME
             value: "gitlab"
           - name: DB_USER
             valueFrom:
               secretKeyRef:
                 name: gitlab-secret
                 key: postgres.user.root
           - name: DB_PASS
             valueFrom:
               secretKeyRef:
                 name: gitlab-secret
                 key: postgres.pwd.root
           - name: REDIS_HOST
             value: "redis"
           - name: REDIS_PORT
             value: "6379"
           ports:
           - containerPort: 80
           resources:
             limits:
               cpu: 2000m
               memory: 5048Mi
             requests:
               cpu: 100m
               memory: 500Mi
           volumeMounts:
           - mountPath: /home/git/data
             name: data
         volumes:
         - name: data
           hostPath:
             path: /var/lib/gitlab/
   
   #部署到k8s-slave2节点
   $ kubectl label node k8s-slave2 gitlab=true
   
   # 创建
   $ kubectl create -f gitlab.yaml
   
   
   
   mkdir -vp gitlab/{data,logs,config}
   sudo docker pull gitlab/gitlab-ce
   
   sudo docker run --detach \
   --hostname 172.16.239.5 \
   --publish 172.16.239.5:443:443 --publish 172.16.239.5:80:80 --publish 172.16.239.5:1022:22 \
   --name gitlab \
   --restart always \
   --volume /root/gitlab/config:/etc/gitlab \
   --volume /root/gitlab/logs:/var/log/gitlab \
   --volume /root/gitlab/data:/var/opt/gitlab \
   gitlab/gitlab-ce:latest
   ```
   
   

配置hosts解析：

```powershell
172.21.51.67 gitlab.luffy.com
```



推送代码到gitlab上，触发jenkins的任务

*设置root密码*

访问http://gitlab.luffy.com，设置管理员密码

*配置k8s-master节点的hosts*

```powershell
$ echo "172.21.51.67 gitlab.luffy.com" >>/etc/hosts
```



*myblog项目推送到gitlab*

```powershell
mkdir demo
cp -r myblog demo/
cd demo/myblog
git remote rename origin old-origin
git remote add origin http://gitlab.luffy.com/root1/myblog.git
git push -u origin --all
git push -u origin --tags


git remote -v
git remote rename origin old-origin
git remote add origin http://172.16.239.7:31296/root2/myblog.git
git push -u origin --all
git push -u origin --tags

1564112138@qq.com
12345678
 
 
```

*钉钉推送*

[官方文档](https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq)

- 配置机器人

- 试验发送消息

  ```powershell
  $ curl 'https://oapi.dingtalk.com/robot/send?access_token=67e81175c6ebacb1307e83f62680f36fbcf4524e8f43971cf2fb2049bc58723d' \
     -H 'Content-Type: application/json' \
     -d '{"msgtype": "text", 
          "text": {
               "content": "我就是我, 是不一样的烟火"
          }
        }'
  
  ```

###### 演示过程

流程示意图：

![](images\jenkins-gitlab.png)

1. 安装gitlab plugin

   插件中心搜索并安装gitlab，直接安装即可

2. 配置Gitlab

   系统管理->系统配置->Gitlab，其中的API Token，需要从下个步骤中获取

   ![](images\gitlab-connection.jpg)

3. 获取AccessToken

   登录gitlab，选择user->Settings->access tokens新建一个访问token

4. 配置host解析

   由于我们的Jenkins和gitlab域名是本地解析，因此需要让gitlab和Jenkins服务可以解析到对方的域名。两种方式：

   - 在容器内配置hosts

   - 配置coredns的静态解析

     ```powershell
             hosts {
                 172.21.51.67 jenkins.luffy.com  gitlab.luffy.com
                 fallthrough
             }
     ```

5. Jenkins创建自由风格项目

   - gitlab connection 选择为刚创建的gitlab
   - 源码管理选择Git，填项项目地址
   - 新建一个 Credentials 认证，使用用户名密码方式，配置gitlab的用户和密码
   - 构建触发器选择 Build when a change is pushed to GitLab 
   - 生成一个Secret token
   - 保存

6. 到gitlab配置webhook

   - 进入项目下settings->Integrations
   - URL： http://jenkins.luffy.com/project/free 
   - Secret Token 填入在Jenkins端生成的token
   - Add webhook
   - test push events，报错：Requests to the local network are not allowed 

7. 设置gitlab允许向本地网络发送webhook请求

   访问 Admin Aera -> Settings -> Network ，展开Outbound requests

   Collapse，勾选第一项即可。再次test push events，成功。

   ![](images\gitlab-webhook-success.jpg)

8. 配置free项目，增加构建步骤，执行shell，将发送钉钉消息的shell保存

9. 提交代码到gitlab仓库，查看构建是否自动执行

> 

##### Master-Slaves（agent）模式

上面演示的任务，默认都是在master节点执行的，多个任务都在master节点执行，对master节点的性能会造成一定影响，如何将任务分散到不同的节点，做成多slave的方式？

1. 添加slave节点

   - 系统管理 -> 节点管理 -> 新建节点
   - 比如添加172.21.51.68，选择固定节点，保存
   - 远程工作目录/opt/jenkins_jobs
   - 标签为任务选择节点的依据，如172.21.51.68
   - 启动方式选择通过java web启动代理，代理是运行jar包，通过JNLP（是一种允许客户端启动托管在远程Web服务器上的应用程序的协议 ）启动连接到master节点服务中

   ![](images\jenkins-new-node.jpg)

2. 执行java命令启动agent服务

   ```powershell
   ## 登录172.21.51.68，下载agent.jar
   $ wget http://jenkins.luffy.com/jnlpJars/agent.jar
   ## 会提示找不到agent错误，因为没有配置地址解析，由于连接jenkins master会通过50000端口，直接使用cluster-ip
   $ kubectl -n jenkins get svc #在master节点执行查询cluster-ip地址
   NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)              AGE
   jenkins   ClusterIP   10.99.204.208   <none>        8080/TCP,50000/TCP   4h8m
   
   ## 再次回到68节点
   $ wget 10.99.204.208:8080/jnlpJars/agent.jar
   $ java -jar agent.jar -jnlpUrl http://10.99.204.208:8080/computer/172.21.51.68/slave-agent.jnlp -secret 4be4d164f861d2830835653567867a1e695b30c320d35eca2be9f5624f8712c8 -workDir "/opt/jenkins_jobs"
   ...
   INFO: Remoting server accepts the following protocols: [JNLP4-connect, Ping]
   Apr 01, 2020 7:03:51 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Agent discovery successful
     Agent address: 10.99.204.208
     Agent port:    50000
     Identity:      e4:46:3a:de:86:24:8e:15:09:13:3d:a7:4e:07:04:37
   Apr 01, 2020 7:03:51 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Handshaking
   Apr 01, 2020 7:03:51 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Connecting to 10.99.204.208:50000
   Apr 01, 2020 7:03:51 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Trying protocol: JNLP4-connect
   Apr 01, 2020 7:04:02 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Remote identity confirmed: e4:46:3a:de:86:24:8e:15:09:13:3d:a7:4e:07:04:37
   Apr 01, 2020 7:04:03 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Connected
   ```

   若出现如下错误:

   ```powershell
   SEVERE: http://jenkins.luffy.com/tcpSlaveAgentListener/ appears to be publishing an invalid X-Instance-Identity.
   java.io.IOException: http://jenkins.luffy.com/tcpSlaveAgentListener/ appears to be publishing an invalid X-Instance-Identity.
           at org.jenkinsci.remoting.engine.JnlpAgentEndpointResolver.resolve(JnlpAgentEndpointResolver.java:287)
           at hudson.remoting.Engine.innerRun(Engine.java:694)
           at hudson.remoting.Engine.run(Engine.java:519)
   ```

   可以选择： 配置从节点 -> 高级 -> Tunnel连接位置，参考下图进行设置:

   ![](images\slave-tunnel.jpg)

3. 查看Jenkins节点列表，新节点已经处于可用状态

   ![](images\jenkins-node-lists.jpg)

4. 测试使用新节点执行任务

   - 配置free项目

   - 限制项目的运行节点 ，标签表达式选择172.21.51.68

   - 立即构建

   - 查看构建日志

     ```powershell
     Started by user admin
     Running as SYSTEM
     Building remotely on 172.21.51.68 in workspace /opt/jenkins_jobs/workspace/free-demo
     using credential gitlab-user
     Cloning the remote Git repository
     Cloning repository http://gitlab.luffy.com/root/myblog.git
      > git init /opt/jenkins_jobs/workspace/free-demo # timeout=10
      ...
     
     ```

> 

##### Jenkins定制化容器

由于每次新部署Jenkins环境，均需要安装很多必要的插件，因此考虑把插件提前做到镜像中

*Dockerfile*

```dockerfile
FROM jenkinsci/blueocean:1.23.2
LABEL maintainer="inspur_lyx@hotmail.com"

## 用最新的插件列表文件替换默认插件文件
COPY plugins.txt /usr/share/jenkins/ref/

## 执行插件安装
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

```

*plugins.txt*

```python
ace-editor:1.1
allure-jenkins-plugin:2.28.1
ant:1.10
antisamy-markup-formatter:1.6
apache-httpcomponents-client-4-api:4.5.10-1.0
authentication-tokens:1.3
...

```

*get_plugin.sh*

> admin:123456@localhost 需要替换成Jenkins的用户名、密码及访问地址

```powershell
#!/usr/bin/env bash
curl -sSL  "http://admin:123456@localhost:8080/pluginManager/api/xml?depth=1&xpath=/*/*/shortName|/*/*/version&wrapper=plugins" | perl -pe 's/.*?<shortName>([\w-]+).*?<version>([^<]+)()(<\/\w+>)+/\1:\2\n/g'|sed 's/ /:/' > plugins.txt

```

```powershell
## 执行构建，定制jenkins容器
$ docker build . -t 172.21.51.67:5000/jenkins:v20200414 -f Dockerfile
$ docker push 172.21.51.67:5000/jenkins:v20200414

```

至此，我们可以使用定制化的镜像启动jenkins服务

```powershell
## 删掉当前服务
$ kubectl delete -f jenkins-all.yaml

## 删掉已挂载的数据
$ rm -rf /var/jenkins_home

## 替换使用定制化镜像
$ sed -i 's#jenkinsci/blueocean#172.21.51.67:5000/jenkins:v20200404#g' jenkins-all.yaml

## 重新创建服务
$ kubectl create -f jenkins-all.yaml

```



##### 本章小结

自由风格项目弊端：

- 任务的完成需要在Jenkins端维护大量的配置
- 没法做版本控制
- 可读性、可移植性很差，不够优雅

> 

#### 流水线入门

![](images\pipeline-factory.jpeg)

[官方文档](https://jenkins.io/zh/doc/book/pipeline/getting-started/)

![](images\realworld-pipeline-flow.png)

为什么叫做流水线，和工厂产品的生产线类似，pipeline是从源码到发布到线上环境。关于流水线，需要知道的几个点：

- 重要的功能插件，帮助Jenkins定义了一套工作流框架；

- Pipeline 的实现方式是一套 Groovy DSL（ 领域专用语言 ），所有的发布流程都可以表述为一段 Groovy 脚本；

- 将WebUI上需要定义的任务，以脚本代码的方式表述出来；

- 帮助jenkins实现持续集成CI（Continue Integration）和持续部署CD（Continue Deliver）的重要手段；

##### 流水线基础语法

[官方文档](https://jenkins.io/zh/doc/book/pipeline/syntax/)

两种语法类型：

- Scripted Pipeline，脚本式流水线，最初支持的类型
- Declarative Pipeline，声明式流水线，为Pipeline plugin在2.5版本之后新增的一种脚本类型，后续Open Blue Ocean所支持的类型。与原先的Scripted Pipeline一样，都可以用来编写脚本。Declarative Pipeline 是后续Open Blue Ocean所支持的类型，写法简单，支持内嵌Scripted Pipeline代码

*为与BlueOcean脚本编辑器兼容，通常建议使用Declarative Pipeline的方式进行编写,从jenkins社区的动向来看，很明显这种语法结构也会是未来的趋势。*

###### 脚本示例

```json
pipeline { 
    agent {label '172.21.51.68'}
    environment { 
        PROJECT = 'myblog'
    }
    stages {
        stage('Checkout') { 
            steps { 
                checkout scm 
            }
        }
        stage('Build') { 
            steps { 
                sh 'make' 
            }
        }
        stage('Test'){
            steps {
                sh 'make check'
                junit 'reports/**/*.xml' 
            }
        }
        stage('Deploy') {
            steps {
                sh 'make publish'
            }
        }
    }
	post {
        success { 
            echo 'Congratulations!'
        }
		failure { 
            echo 'Oh no!'
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}

```

###### 脚本解释：

- `checkout`步骤为检出代码; `scm`是一个特殊变量，指示`checkout`步骤克隆触发此Pipeline运行的特定修订 

- agent：指明使用哪个agent节点来执行任务，定义于pipeline顶层或者stage内部

  - any，可以使用任意可用的agent来执行

  - label，在提供了标签的 Jenkins 环境中可用的代理上执行流水线或阶段。 例如: `agent { label 'my-defined-label' }`，最常见的使用方式

  - none，当在 `pipeline` 块的顶部没有全局代理， 该参数将会被分配到整个流水线的运行中并且每个 `stage` 部分都需要包含他自己的 `agent` 部分。比如: `agent none`

  - docker， 使用给定的容器执行流水线或阶段。 在指定的节点中，通过运行容器来执行任务

    ```json
    agent {
        docker {
            image 'maven:3-alpine'
            label 'my-defined-label'
            args  '-v /tmp:/tmp'
        }
    }
    
    ```

- options:  允许从流水线内部配置特定于流水线的选项。  

  -  buildDiscarder , 为最近的流水线运行的特定数量保存组件和控制台输出。例如: `options { buildDiscarder(logRotator(numToKeepStr: '10')) }` 
  -  disableConcurrentBuilds ,不允许同时执行流水线。 可被用来防止同时访问共享资源等。 例如: `options { disableConcurrentBuilds() }`
  -  timeout ,设置流水线运行的超时时间, 在此之后，Jenkins将中止流水线。例如: `options { timeout(time: 1, unit: 'HOURS') }`
  -  retry，在失败时, 重新尝试整个流水线的指定次数。 For example: `options { retry(3) }`

- environment:  指令制定一个 键-值对序列，该序列将被定义为所有步骤的环境变量 

- stages: 包含一系列一个或多个 [stage](https://jenkins.io/zh/doc/book/pipeline/syntax/#stage)指令, `stages` 部分是流水线描述的大部分"work" 的位置。 建议 `stages` 至少包含一个 [stage](https://jenkins.io/zh/doc/book/pipeline/syntax/#stage) 指令用于连续交付过程的每个离散部分,比如构建, 测试, 和部署。 

  ```groovy
  pipeline {
      agent any
      stages { 
          stage('Example') {
              steps {
                  echo 'Hello World'
              }
          }
      }
  }
  
  ```

- steps: 在给定的 `stage` 指令中执行的定义了一系列的一个或多个[steps](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-steps)。

- post:  定义一个或多个[steps](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-steps) ，这些阶段根据流水线或阶段的完成情况而运行`post` 支持以下 [post-condition](https://jenkins.io/zh/doc/book/pipeline/syntax/#post-conditions) 块中的其中之一: `always`, `changed`, `failure`, `success`, `unstable`, 和 `aborted`。

  -  always, 无论流水线或阶段的完成状态如何，都允许在 `post` 部分运行该步骤
  -  changed, 当前流水线或阶段的完成状态与它之前的运行不同时，才允许在 `post` 部分运行该步骤
  -  failure,  当前流水线或阶段的完成状态为"failure"，才允许在 `post` 部分运行该步骤, 通常web UI是红色
  -  success,  当前流水线或阶段的完成状态为"success"，才允许在 `post` 部分运行该步骤, 通常web UI是蓝色或绿色
  -  unstable,  当前流水线或阶段的完成状态为"unstable"，才允许在 `post` 部分运行该步骤, 通常由于测试失败,代码违规等造成。通常web UI是黄色 
  -  aborted， 只有当前流水线或阶段的完成状态为"aborted"，才允许在 `post` 部分运行该步骤, 通常由于流水线被手动的aborted。通常web UI是灰色 

创建pipeline示意：

新建任务 -> 流水线 

`jenkins/pipelines/p1.yaml`

```powershell
pipeline {
   agent {label '172.21.51.68'}
   environment { 
      PROJECT = 'myblog'
   }
   stages {
      stage('printenv') {
         steps {
            echo 'Hello World'
            sh 'printenv'
         }
      }
      stage('check') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'gitlab-user', url: 'http://gitlab.luffy.com/root/myblog.git']]])
         }
      }
      stage('build-image') {
         steps {
            sh 'docker build . -t myblog:latest -f Dockerfile'
         }
      }
      stage('send-msg') {
         steps {
            sh """
            curl 'https://oapi.dingtalk.com/robot/send?access_token=67e81175c6ebacb1307e83f62680f36fbcf4524e8f43971cf2fb2049bc58723d' \
   -H 'Content-Type: application/json' \
   -d '{"msgtype": "text", 
        "text": {
             "content": "我就是我, 是不一样的烟火"
        }
      }'
      """
         }
      }
   }
}

```

点击“立即构建”，同样的，我们可以配置触发器，使用webhook的方式接收项目的push事件，

- 构建触发器选择 Build when a change is pushed to GitLab. 
- 生成 Secret token 
- 配置gitlab，创建webhook，发送test push events测试

###### Blue Ocean:

[官方文档]( https://jenkins.io/zh/doc/book/blueocean/getting-started/ )

我们需要知道的几点：

- 是一个插件， 旨在为Pipeline提供丰富的体验 ；
- 连续交付（CD）Pipeline的复杂可视化，允许快速和直观地了解Pipeline的状态；
- 目前支持的类型仅针对于Pipeline，尚不能替代Jenkins 经典版UI

思考：

1. 每个项目都把大量的pipeline脚本写在Jenkins端，对于谁去维护及维护成本是一个问题
2. 没法做版本控制

> 

##### Jenkinsflie

Jenkins Pipeline 提供了一套可扩展的工具，用于将“简单到复杂”的交付流程实现为“持续交付即代码”。Jenkins Pipeline 的定义通常被写入到一个文本文件（称为 `Jenkinsfile` ）中，该文件可以被放入项目的源代码控制库中。  

###### 演示1：使用Jenkinsfile管理**pipeline** 

- 在项目中新建Jenkinsfile文件，拷贝已有script内容
- 配置pipeline任务，流水线定义为Pipeline Script from SCM
- 执行push 代码测试

Jenkinsfile:

`jenkins/pipelines/p2.yaml`

```powershell
pipeline {
   agent { label '172.21.51.68'}

   stages {
      stage('printenv') {
         steps {
            echo 'Hello World'
            sh 'printenv'
         }
      }
      stage('check') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'gitlab-user', url: 'http://gitlab.luffy.com/root/myblog.git']]])
         }
      }
      stage('build-image') {
         steps {
            retry(2) { sh 'docker build . -t myblog:latest'}
         }
      }
      stage('send-msg') {
         steps {
            sh """
            curl 'https://oapi.dingtalk.com/robot/send?access_token=67e81175c6ebacb1307e83f62680f36fbcf4524e8f43971cf2fb2049bc58723d' \
   -H 'Content-Type: application/json' \
   -d '{"msgtype": "text", 
        "text": {
             "content": "我就是我, 是不一样的烟火"
        }
      }'
      """
         }
      }
   }
}

```

###### 演示2：优化及丰富流水线内容

- 优化代码检出阶段

  由于目前已经配置了使用git仓库地址，且使用SCM来检测项目，因此代码检出阶段完全没有必要再去指定一次 

- 构建镜像的tag使用git的commit id

- 增加post阶段的消息通知，丰富通知内容

- 配置webhook，实现myblog代码推送后，触发Jenkinsfile任务执行

`jenkins/pipelines/p3.yaml`

```groovy
pipeline {
    agent { label '172.21.51.68'}

    stages {
        stage('printenv') {
            steps {
            echo 'Hello World'
            sh 'printenv'
            }
        }
        stage('check') {
            steps {
                checkout scm
            }
        }
        stage('build-image') {
            steps {
            	retry(2) { sh 'docker build . -t myblog:${GIT_COMMIT}'}
            }
        }
    }
    post {
        success { 
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=67e81175c6ebacb1307e83f62680f36fbcf4524e8f43971cf2fb2049bc58723d' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text", 
                            "text": {
                                "content": "😄👍构建成功👍😄\n 关键字：luffy\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=67e81175c6ebacb1307e83f62680f36fbcf4524e8f43971cf2fb2049bc58723d' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text", 
                            "text": {
                                "content": "😖❌构建失败❌😖\n 关键字：luffy\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}

```

###### 演示3：使用k8s部署服务

- 新建deploy目录，将k8s所需的文件放到deploy目录中

- 将镜像地址改成模板，在pipeline中使用新构建的镜像进行替换

- 执行kubectl apply -f deploy应用更改，需要配置kubectl认证

  ```powershell
  $ scp -r k8s-master:/root/.kube /root
  ```
  
  

`jenkins/pipelines/p4.yaml`

```powershell
pipeline {
    agent { label '172.21.51.68'}

    environment {
        IMAGE_REPO = "172.21.51.67:5000/myblog"
    }

    stages {
        stage('printenv') {
            steps {
              echo 'Hello World'
              sh 'printenv'
            }
        }
        stage('check') {
            steps {
                checkout scm
            }
        }
        stage('build-image') {
            steps {
                retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
            }
        }
        stage('push-image') {
            steps {
                retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
            }
        }
        stage('deploy') {
            steps {
                sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                timeout(time: 1, unit: 'MINUTES') {
                    sh "kubectl apply -f deploy/"
                }
            }
        }
    }
    post {
        success { 
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=67e81175c6ebacb1307e83f62680f36fbcf4524e8f43971cf2fb2049bc58723d' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text", 
                            "text": {
                                "content": "😄👍构建成功👍😄\n 关键字：myblog\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=67e81175c6ebacb1307e83f62680f36fbcf4524e8f43971cf2fb2049bc58723d' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text", 
                            "text": {
                                "content": "😖❌构建失败❌😖\n 关键字：luffy\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}

```

###### 演示4：使用凭据管理敏感信息

上述Jenkinsfile中存在的问题是敏感信息使用明文，暴漏在代码中，如何管理流水线中的敏感信息（包含账号密码），之前我们在对接gitlab的时候，需要账号密码，已经使用过凭据来管理这类敏感信息，同样的，我们可以使用凭据来存储钉钉的token信息，那么，创建好凭据后，如何在Jenkinsfile中获取已有凭据的内容？

 Jenkins 的声明式流水线语法有一个 `credentials()` 辅助方法（在[`environment`](https://jenkins.io/zh/doc/book/pipeline/jenkinsfile/#../syntax#environment) 指令中使用），它支持 [secret 文本](https://jenkins.io/zh/doc/book/pipeline/jenkinsfile/##secret-text)，[带密码的用户名](https://jenkins.io/zh/doc/book/pipeline/jenkinsfile/##usernames-and-passwords)，以及 [secret 文件](https://jenkins.io/zh/doc/book/pipeline/jenkinsfile/##secret-files)凭据。 

下面的流水线代码片段展示了如何创建一个使用带密码的用户名凭据的环境变量的流水线。

在该示例中，带密码的用户名凭据被分配了环境变量，用来使你的组织或团队以一个公用账户访问 Bitbucket 仓库；这些凭据已在 Jenkins 中配置了凭据 ID `jenkins-bitbucket-common-creds`。

当在 [`environment`](https://jenkins.io/zh/doc/book/pipeline/jenkinsfile/#../syntax#environment) 指令中设置凭据环境变量时：

```
environment {
    BITBUCKET_COMMON_CREDS = credentials('jenkins-bitbucket-common-creds')
}

```

这实际设置了下面的三个环境变量：

- `BITBUCKET_COMMON_CREDS` - 包含一个以冒号分隔的用户名和密码，格式为 `username:password`。
- `BITBUCKET_COMMON_CREDS_USR` - 附加的一个仅包含用户名部分的变量。
- `BITBUCKET_COMMON_CREDS_PSW` - 附加的一个仅包含密码部分的变量。

```groovy
pipeline {
    agent {
        // 此处定义 agent 的细节
    }
    environment {
        //顶层流水线块中使用的 environment 指令将适用于流水线中的所有步骤。 
        BITBUCKET_COMMON_CREDS = credentials('jenkins-bitbucket-common-creds')
    }
    stages {
        stage('Example stage 1') {
 			//在一个 stage 中定义的 environment 指令只会将给定的环境变量应用于 stage 中的步骤。
            environment {
                BITBUCKET_COMMON_CREDS = credentials('another-credential-id')
            }
            steps {
                // 
            }
        }
        stage('Example stage 2') {
            steps {
                // 
            }
        }
    }
}

```

 因此对Jenkinsfile做改造：

`jenkins/pipelines/p5.yaml`

```powershell
pipeline {
    agent { label '172.21.51.68'}

    environment {
        IMAGE_REPO = "172.21.51.67:5000/myblog"
        DINGTALK_CREDS = credentials('dingTalk')
    }

    stages {
        stage('printenv') {
            steps {
            echo 'Hello World'
            sh 'printenv'
            }
        }
        stage('check') {
            steps {
                checkout scm
            }
        }
        stage('build-image') {
            steps {
                retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
            }
        }
        stage('push-image') {
            steps {
                retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
            }
        }
        stage('deploy') {
            steps {
                sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                timeout(time: 1, unit: 'MINUTES') {
                    sh "kubectl apply -f deploy/"
                }
            }
        }
    }
    post {
        success { 
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text", 
                            "text": {
                                "content": "😄👍构建成功👍😄\n 关键字：luffy\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text", 
                            "text": {
                                "content": "😖❌构建失败❌😖\n 关键字：luffy\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}

```

###### 本章小结

上面我们已经通过Jenkinsfile完成了最简单的项目的构建和部署，那么我们来思考目前的方式：

1. 目前都是在项目的单一分支下进行操作，企业内一般会使用feature、develop、release、master等多个分支来管理整个代码提交流程，如何根据不同的分支来做构建？
2. 构建视图中如何区分不同的分支?
3. 如何不配置webhook的方式实现构建？
4. 如何根据不同的分支选择发布到不同的环境(开发、测试、生产)？

> 

##### 多分支流水线

[官方示例]( https://jenkins.io/zh/doc/tutorials/build-a-multibranch-pipeline-project/ )

我们简化一下流程，假如使用develop分支作为开发分支，master分支作为集成测试分支，看一下如何使用多分支流水线来管理。

###### 演示1：多分支流水线的使用

1. 提交develop分支：

```powershell
$ git checkout -b develop
$ git push --set-upstream origin develop

```

2. 禁用pipeline项目

3. Jenkins端创建多分支流水线项目
   - 增加git分支源
   - 发现标签
   - 根据名称过滤，develop|master|v.*
   - 高级克隆，设置浅克隆

保存后，会自动检索项目中所有存在Jenkinsfile文件的分支和标签，若匹配我们设置的过滤正则表达式，则会添加到多分支的构建视图中。所有添加到视图中的分支和标签，会默认执行一次构建任务。

###### 演示2：美化消息通知内容

- 添加构建阶段记录
- 使用markdown格式，添加构建分支消息

`jenkins/pipelines/p6.yaml`

```powershell
pipeline {
    agent { label '172.21.51.68'}

    environment {
        IMAGE_REPO = "172.21.51.67:5000/myblog"
        DINGTALK_CREDS = credentials('dingTalk')
        TAB_STR = "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
    }

    stages {
        stage('printenv') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        }
        stage('checkout') {
            steps {
                checkout scm
                script{
                    env.BUILD_TASKS = env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('build-image') {
            steps {
                retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('push-image') {
            steps {
                retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('deploy') {
            steps {
                sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                timeout(time: 1, unit: 'MINUTES') {
                    sh "kubectl apply -f deploy/"
                }
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
    }
    post {
        success { 
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${GIT_BRANCH}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """ 
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😖❌ 构建失败 ❌😖  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${GIT_BRANCH}  \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}

```

###### 演示3：通知gitlab构建状态

Jenkins端做了构建，可以通过gitlab通过的api将构建状态通知过去，作为开发人员发起Merge Request或者合并Merge Request的依据之一。

*注意一定要指定gitLabConnection('gitlab')，不然没法认证到Gitlab端*

`jenkins/pipelines/p7.yaml`

```powershell
pipeline {
    agent { label '172.21.51.68'}
    
    options {
		buildDiscarder(logRotator(numToKeepStr: '10'))
		disableConcurrentBuilds()
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}

    environment {
        IMAGE_REPO = "172.21.51.67:5000/demo/myblog"
        DINGTALK_CREDS = credentials('dingTalk')
        TAB_STR = "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
    }

    stages {
        stage('printenv') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        }
        stage('checkout') {
            steps {
                checkout scm
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS = env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('build-image') {
            steps {
                retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('push-image') {
            steps {
                retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('deploy') {
            steps {
                sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                timeout(time: 1, unit: 'MINUTES') {
                    sh "kubectl apply -f deploy/"
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
    }
    post {
        success { 
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """ 
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😖❌ 构建失败 ❌😖  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}  \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}

```

我们可以访问gitlab，然后找到commit记录，查看同步状态

![](images\gitlab-cicd.jpg)

提交merge request，也可以查看到相关的任务状态，可以作为项目owner合并代码的依据之一：

![](images\gitlab-merge-request.jpg)



###### 本章小节

优势:

- 根据分支展示, 视图人性化
- 自动检测各分支的变更

思考：

- Jenkins的slave端，没有任务的时候处于闲置状态，slave节点多的话造成资源浪费
- 是否可以利用kubernetes的Pod来启动slave，动态slave pod来执行构建任务

> 

#### 工具集成与Jenkinsfile实践篇

1. Jenkins如何对接kubernetes集群
2. 使用kubernetes的Pod-Template来作为动态的agent执行Jenkins任务
3. 如何制作agent容器实现不同类型的业务的集成
4. 集成代码扫描、docker镜像自动构建、k8s服务部署、自动化测试

##### 集成Kubernetes

###### 插件安装及配置

[插件官方文档](https://plugins.jenkins.io/kubernetes/)

1. [系统管理] -> [插件管理] -> [搜索kubernetes]->直接安装

   若安装失败，请先更新[ bouncycastle API Plugin](https://plugins.jenkins.io/bouncycastle-api)并重新启动Jenkins

2. [系统管理] -> [系统配置] -> [Add a new cloud]

3. 配置地址信息

   -  Kubernetes 地址: https://kubernetes.default（或者https://172.21.51.67:6443）
   -  Kubernetes 命名空间：jenkins
   -  服务证书不用写（我们在安装Jenkins的时候已经指定过serviceAccount），均使用默认
   -  连接测试，成功会提示：Connection test successful
   -  Jenkins地址：http://jenkins:8080
   -  Jenkins 通道 ：jenkins:50000

4. 配置Pod Template

   - 名称：jnlp-slave
   - 命名空间：jenkins
   - 标签列表：jnlp-slave，作为agent的label选择用
   - 连接 Jenkins 的超时时间（秒） ：300，设置连接jenkins超时时间
   - 节点选择器：agent=true
   - 工作空间卷：选择hostpath，设置/opt/jenkins_jobs/,注意需要设置chown -R 1000:1000 /opt/jenkins_jobs/权限，否则Pod没有权限

###### 演示动态slave pod

```powershell
# 为准备运行jnlp-slave-agent的pod的节点打上label
$ kubectl label node k8s-slave1 agent=true

### 回放一次多分支流水线develop分支
agent { label 'jnlp-slave'}

```

执行任务，会下载默认的jnlp-slave镜像，地址为jenkins/inbound-agent:4.3-4，我们可以先在k8s-master节点拉取下来该镜像：

```powershell
$ docker pull jenkins/inbound-agent:4.3-4

```

保存jenkinsfile提交后，会出现报错，因为我们的agent已经不再是宿主机，而是Pod中的容器内，报错如下：

![](images\gitlab-no-docker-err.png)

因此我们需要将用到的命令行工具集成到Pod的容器内，但是思考如下问题：

- 目前是用的jnlp的容器，是java的环境，我们在此基础上需要集成很多工具，能不能创建一个新的容器，让新容器来做具体的任务，jnlp-slave容器只用来负责连接jenkins-master
- 针对不同的构建环境（java、python、go、nodejs），可以制作不同的容器，来执行对应的任务

> 

###### Pod-Template中容器镜像的制作

为解决上述问题，我们制作一个tools镜像，集成常用的工具，来完成常见的构建任务，需要注意的几点：

- 使用alpine基础镜像，自身体积比较小
- 替换国内安装源
- 为了使用docker，安装了docker
- 为了克隆代码，安装git
- 为了后续做python的测试等任务，安装python环境
- 为了在容器中调用kubectl的命令，拷贝了kubectl的二进制文件
- 为了认证kubectl，需要在容器内部生成.kube目录及config文件

```powershell
$ mkdir tools;
$ cd tools;
$ cp `which kubectl` .
$ cp ~/.kube/config .

```

*Dockerfile*

`jenkins/custom-images/tools/Dockerfile`

```dockerfile
FROM alpine
LABEL maintainer="inspur_lyx@hotmail.com"
USER root

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && \
    apk update && \
    apk add  --no-cache openrc docker git curl tar gcc g++ make \
    bash shadow openjdk8 python2 python2-dev py-pip python3-dev openssl-dev libffi-dev \
    libstdc++ harfbuzz nss freetype ttf-freefont && \
    mkdir -p /root/.kube && \
    usermod -a -G docker root

COPY config /root/.kube/

RUN rm -rf /var/cache/apk/* 
#-----------------安装 kubectl--------------------#
COPY kubectl /usr/local/bin/
RUN chmod +x /usr/local/bin/kubectl
# ------------------------------------------------#

```

执行镜像构建并推送到仓库中：

```powershell
$ docker build . -t 172.21.51.67:5000/devops/tools:v1
$ docker push 172.21.51.67:5000/devops/tools:v1

```

我们可以直接使用该镜像做测试：

```powershell
## 启动临时镜像做测试
$ docker run --rm -ti 172.21.51.67:5000/devops/tools:v1 bash
# / git clone http://xxxxxx.git
# / kubectl get no
# / python3
#/ docker

## 重新挂载docker的sock文件
docker run -v /var/run/docker.sock:/var/run/docker.sock --rm -ti 172.21.51.67:5000/devops/tools:v1 bash

```



###### 实践通过Jenkinsfile实现demo项目自动发布到kubenetes环境

更新Jenkins中的PodTemplate，添加tools镜像，注意同时要先添加名为jnlp的container，因为我们是使用自定义的PodTemplate覆盖掉默认的模板：

![](images\pod-template-jnlp.jpg)

在卷栏目，添加卷，Host Path Volume，不然在容器中使用docker会提示docker服务未启动

![](images\jenkins-docker-sock.png)



tools容器做好后，我们需要对Jenkinsfile做如下调整：

`jenkins/pipelines/p8.yaml`

```powershell
pipeline {
    agent { label 'jnlp-slave'}
    
    options {
		buildDiscarder(logRotator(numToKeepStr: '10'))
		disableConcurrentBuilds()
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}

    environment {
        IMAGE_REPO = "172.21.51.67:5000/myblog"
        DINGTALK_CREDS = credentials('dingTalk')
        TAB_STR = "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
    }

    stages {
        stage('printenv') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        }
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS = env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('build-image') {
            steps {
                container('tools') {
                    retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('push-image') {
            steps {
                container('tools') {
                    retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('deploy') {
            steps {
                container('tools') {
                    sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                    timeout(time: 1, unit: 'MINUTES') {
                        sh "kubectl apply -f deploy/"
                    }
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
    }
    post {
        success { 
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """ 
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😖❌ 构建失败 ❌😖  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}  \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}

```

> 

##### 集成sonarQube实现代码扫描

Sonar可以从以下七个维度检测代码质量，而作为开发人员至少需要处理前5种代码质量问题。

1. 不遵循代码标准
   sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具规范代码编写。
2. 潜在的缺陷
   sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具检 测出潜在的缺陷。
3. 糟糕的复杂度分布
   文件、类、方法等，如果复杂度过高将难以改变，这会使得开发人员 难以理解它们, 且如果没有自动化的单元测试，对于程序中的任何组件的改变都将可能导致需要全面的回归测试。
4. 重复
   显然程序中包含大量复制粘贴的代码是质量低下的，sonar可以展示 源码中重复严重的地方。
5. 注释不足或者过多
   没有注释将使代码可读性变差，特别是当不可避免地出现人员变动 时，程序的可读性将大幅下降 而过多的注释又会使得开发人员将精力过多地花费在阅读注释上，亦违背初衷。
6. 缺乏单元测试
   sonar可以很方便地统计并展示单元测试覆盖率。
7. 糟糕的设计
   通过sonar可以找出循环，展示包与包、类与类之间的相互依赖关系，可以检测自定义的架构规则 通过sonar可以管理第三方的jar包，可以利用LCOM4检测单个任务规则的应用情况， 检测耦合。

###### sonarqube架构简介

![](images\sonarqube.webp)

1. CS架构
   - sonarqube scanner
   - sonarqube server
2. SonarQube Scanner 扫描仪在本地执行代码扫描任务
3. 执行完后，将分析报告被发送到SonarQube服务器进行处理
4. SonarQube服务器处理和存储分析报告导致SonarQube数据库，并显示结果在UI中

###### sonarqube on kubernetes环境搭建

1. 资源文件准备

`sonar/sonar.yaml`

- 和gitlab共享postgres数据库
- 使用ingress地址 `sonar.luffy.com` 进行访问
- 使用initContainers进行系统参数调整

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sonarqube
  namespace: jenkins
  labels:
    app: sonarqube
spec:
  ports:
  - name: sonarqube
    port: 9000
    targetPort: 9000
    protocol: TCP
  selector:
    app: sonarqube
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: jenkins
  name: sonarqube
  labels:
    app: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      nodeSelector:
        sonar: "true"
      initContainers:
      - command:
        - /sbin/sysctl
        - -w
        - vm.max_map_count=262144
        image: alpine:3.6
        imagePullPolicy: IfNotPresent
        name: elasticsearch-logging-init
        resources: {}
        securityContext:
          privileged: true
      containers:
      - name: sonarqube
        image: 172.21.51.67:5000/sonarqube:7.9-community
        ports:
        - containerPort: 9000
        env:
        - name: SONARQUBE_JDBC_USERNAME
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.user.root
        - name: SONARQUBE_JDBC_PASSWORD
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.pwd.root
        - name: SONARQUBE_JDBC_URL
          value: "jdbc:postgresql://postgres:5432/sonar"
        livenessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 6
        resources:
          limits:
            cpu: 2000m
            memory: 4096Mi
          requests:
            cpu: 300m
            memory: 512Mi
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: sonarqube
  namespace: jenkins
spec:
  rules:
  - host: sonar.luffy.com
    http:
      paths:
      - backend:
          serviceName: sonarqube
          servicePort: 9000
        path: /
status:
  loadBalancer: {}

```

2. sonarqube服务端安装

   ```powershell
   
   # 创建sonar数据库
   $ kubectl -n jenkins exec -ti postgres-5859dc6f58-mgqz9 bash
   #/ psql 
   # create database sonar;
   
   ## 创建sonarqube服务器
   $ kubectl create -f sonar.yaml
   
   ## 配置本地hosts解析
   172.21.51.67 sonar.luffy.com
   
   ## 访问sonarqube，初始用户名密码为 admin/admin
   $ curl http://sonar.luffy.com
   
   ```
   
3. sonar-scanner的安装

   下载地址： https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.2.0.1873-linux.zip。该地址比较慢，可以在网盘下载（https://pan.baidu.com/s/1SiEhWyHikTiKl5lEMX1tJg 提取码: tqb9）。

4. 演示sonar代码扫描功能

   - 在项目根目录中准备配置文件 **sonar-project.properties** 

     ```powershell
     sonar.projectKey=myblog
     sonar.projectName=myblog
     # if you want disabled the DTD verification for a proxy problem for example, true by default
     sonar.coverage.dtdVerification=false
     # JUnit like test report, default value is test.xml
     sonar.sources=blog,myblog
     
     ```

   - 配置sonarqube服务器地址

     由于sonar-scanner需要将扫描结果上报给sonarqube服务器做质量分析，因此我们需要在sonar-scanner中配置sonarqube的服务器地址：

     在集群宿主机中测试，先配置一下hosts文件，然后配置sonar的地址：

     ```powershell
     $ cat /etc/hosts
     172.21.51.67  sonar.luffy.com
     
     $ cat sonar-scanner/conf/sonar-scanner.properties
     #----- Default SonarQube server
     #sonar.host.url=http://localhost:9000
     sonar.host.url=http://sonar.luffy.com
     #----- Default source code encoding
     #sonar.sourceEncoding=UTF-8
     
     ```
  ```
   
- 为了使所有的pod都可以通过`sonar.luffy.com`访问，可以配置coredns的静态解析
   
     ```powershell
             hosts {
                 172.21.51.67 jenkins.luffy.com gitlab.luffy.com sonar.luffy.com
                 fallthrough
          }
  ```

   - 执行扫描
   
     ```powershell
     ## 在项目的根目录下执行
     $ /opt/sonar-scanner-4.0.0.1744-linux/bin/sonar-scanner  -X 
     
     ```
   
   - sonarqube界面查看结果
   
     登录sonarqube界面查看结果，Quality Gates说明



###### 插件安装及配置

1. 集成到tools容器中

   由于我们的代码拉取、构建任务均是在tools容器中进行，因此我们需要把scanner集成到我们的tools容器中，又因为scanner是一个cli客户端，因此我们直接把包解压好，拷贝到tools容器内部，配置一下PATH路径即可，注意两点：

   - 直接在在tools镜像中配置`http://sonar.luffy.com`

   - 由于tools已经集成了java环境，因此可以直接剔除scanner自带的jre

     - 删掉sonar-scanner/jre目录

     - 修改sonar-scanner/bin/sonar-scanner

       `use_embedded_jre=false`

   ```powershell
   $ cd tools
   $ cp -r /opt/sonar-scanner-4.0.0.1744-linux/ sonar-scanner
   ## sonar配置，由于我们是在Pod中使用，也可以直接配置：sonar.host.url=http://sonarqube:9000
   $ cat sonar-scanner/conf/sonar-scanner.properties
   #----- Default SonarQube server
   sonar.host.url=http://sonar.luffy.com
   
   #----- Default source code encoding
   #sonar.sourceEncoding=UTF-8
   
   $ rm -rf sonar-scanner/jre
   $ vi sonar-scanner/bin/sonar-scanner
   ...
   use_embedded_jre=false
   ...
   
   ```

   *Dockerfile*

   `jenkins/custom-images/tools/Dockerfile2`

   ```dockerfile
   FROM alpine
   LABEL maintainer="inspur_lyx@hotmail.com"
   USER root
   
   RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && \
       apk update && \
       apk add  --no-cache openrc docker git curl tar gcc g++ make \
       bash shadow openjdk8 python2 python2-dev py-pip python3-dev openssl-dev libffi-dev \
       libstdc++ harfbuzz nss freetype ttf-freefont && \
       mkdir -p /root/.kube && \
       usermod -a -G docker root
   
   COPY config /root/.kube/
   
   
   RUN rm -rf /var/cache/apk/*
   
   #-----------------安装 kubectl--------------------#
   COPY kubectl /usr/local/bin/
   RUN chmod +x /usr/local/bin/kubectl
   # ------------------------------------------------#
   
   #---------------安装 sonar-scanner-----------------#
   COPY sonar-scanner /usr/lib/sonar-scanner
   RUN ln -s /usr/lib/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner && chmod +x /usr/local/bin/sonar-scanner
   ENV SONAR_RUNNER_HOME=/usr/lib/sonar-scanner
   # ------------------------------------------------#
   
   ```

重新构建镜像，并推送到仓库：

```powershell
   $ docker build . -t 172.21.51.67:5000/devops/tools:v2
   $ docker push 172.21.51.67:5000/devops/tools:v2
   
```

2. 修改Jenkins PodTemplate

   为了在新的构建任务中可以拉取v2版本的tools镜像，需要更新PodTemplate

3. 安装并配置sonar插件

   由于sonarqube的扫描的结果需要进行Quality Gates的检测，那么我们在容器中执行完代码扫描任务后，如何知道本次扫描是否通过了Quality Gates，那么就需要借助于sonarqube实现的jenkins的插件。

   - 安装插件

     插件中心搜索sonarqube，直接安装

   - 配置插件

     系统管理->系统配置-> **SonarQube servers** ->Add SonarQube

     - Name：sonarqube

     - Server URL：http://sonar.luffy.com

     - Server authentication token

       ① 登录sonarqube -> My Account -> Security -> Generate Token

       ② 登录Jenkins，添加全局凭据，类型为Secret text

   - 如何在jenkinsfile中使用

     我们在 https://jenkins.io/doc/pipeline/steps/sonar/ 官方介绍中可以看到：

     


###### Jenkinsfile集成sonarqube演示

`jenkins/pipelines/p9.yaml`

```groovy
pipeline {
    agent { label 'jnlp-slave'}
    
    options {
		buildDiscarder(logRotator(numToKeepStr: '10'))
		disableConcurrentBuilds()
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}

    environment {
        IMAGE_REPO = "172.21.51.67:5000/myblog"
        DINGTALK_CREDS = credentials('dingTalk')
        TAB_STR = "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
    }

    stages {
        stage('git-log') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        }        
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS = env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('CI'){
            failFast true
            parallel {
                stage('Unit Test') {
                    steps {
                        echo "Unit Test Stage Skip..."
                    }
                }
                stage('Code Scan') {
                    steps {
                        container('tools') {
                            withSonarQubeEnv('sonarqube') {
                                sh 'sonar-scanner -X'
                                sleep 3
                            }
                            script {
                                timeout(1) {
                                    def qg = waitForQualityGate('sonarqube')
                                    if (qg.status != 'OK') {
                                        error "未通过Sonarqube的代码质量阈检查，请及时修改！failure: ${qg.status}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('build-image') {
            steps {
                container('tools') {
                    retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('push-image') {
            steps {
                container('tools') {
                    retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('deploy') {
            steps {
                container('tools') {
                    sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                    timeout(time: 1, unit: 'MINUTES') {
                        sh "kubectl apply -f deploy/"
                    }
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
    }
    post {
        success { 
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """ 
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😖❌ 构建失败 ❌😖  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}  \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}
```

> 

##### 集成RobotFramework实现验收测试

一个基于Python语言，用于验收测试和验收测试驱动开发（ATDD）的通用测试自动化框架，提供了一套特定的语法，并且有非常丰富的测试库 。



###### robot用例简介

`robot/robot.txt`

```powershell
*** Settings ***
Library           RequestsLibrary
Library           SeleniumLibrary

*** Variables ***
${demo_url}       http://myblog.luffy/admin

*** Test Cases ***
api
    [Tags]  critical
    Create Session    api    ${demo_url}
    ${alarm_system_info}    RequestsLibrary.Get Request    api    /
    log    ${alarm_system_info.status_code}
    log    ${alarm_system_info.content}
    should be true    ${alarm_system_info.status_code} == 200

ui
    [Tags]  critical
    ${chrome_options} =     Evaluate    sys.modules['selenium.webdriver'].ChromeOptions()    sys, selenium.webdriver
    Call Method    ${chrome_options}   add_argument    headless
    Call Method    ${chrome_options}   add_argument    no-sandbox
    ${options}=     Call Method     ${chrome_options}    to_capabilities
    Open Browser    ${demo_url}/    browser=chrome       desired_capabilities=${options}
    sleep    2s
    Capture Page Screenshot
    Page Should Contain    Django
    close browser

```

```powershell
# 使用tools镜像启动容器，来验证手动使用robotframework来做验收测试
$ docker run --rm -ti 172.21.51.67:5000/devops/tools:v2 bash
bash-5.0# apk add chromium chromium-chromedriver
$ cat requirements.txt
robotframework
robotframework-seleniumlibrary
robotframework-databaselibrary
robotframework-requests

#pip安装必要的软件包
$ pip install -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com -r requirements.txt 

#使用robot命令做测试
$ robot -d artifacts/ robot.txt
```

###### 与tools工具镜像集成

```powershell
FROM alpine
LABEL maintainer="inspur_lyx@hotmail.com"
USER root

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && \
    apk update && \
    apk add  --no-cache openrc docker git curl tar gcc g++ make \
    bash shadow openjdk8 python2 python2-dev py-pip python3-dev openssl-dev libffi-dev \
    libstdc++ harfbuzz nss freetype ttf-freefont chromium chromium-chromedriver && \
    mkdir -p /root/.kube && \
    usermod -a -G docker root


COPY config /root/.kube/

COPY requirements.txt /

RUN pip install -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com -r requirements.txt 


RUN rm -rf /var/cache/apk/* && \
    rm -rf ~/.cache/pip

#-----------------安装 kubectl--------------------#
COPY kubectl /usr/local/bin/
RUN chmod +x /usr/local/bin/kubectl
# ------------------------------------------------#

#---------------安装 sonar-scanner-----------------#
COPY sonar-scanner /usr/lib/sonar-scanner
RUN ln -s /usr/lib/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner && chmod +x /usr/local/bin/sonar-scanner
ENV SONAR_RUNNER_HOME=/usr/lib/sonar-scanner
# ------------------------------------------------#

```



```powershell
$ docker build . -t 172.21.51.67:5000/devops/tools:v3

$ docker push 172.21.51.67:5000/devops/tools:v3
```



更新Jenkins中kubernetes中的containers template



###### 插件安装及配置

为什么要安装robot插件？

1. 安装robotFramework

   - 插件中心搜索robotframework，直接安装
   - tools集成robot命令（之前已经安装）

2. 与jenkinsfile的集成

   ```groovy
       container('tools') {
           sh 'robot -i critical  -d artifacts/ robot.txt || echo ok'
           echo "R ${currentBuild.result}"
           step([
               $class : 'RobotPublisher',
               outputPath: 'artifacts/',
               outputFileName : "output.xml",
               disableArchiveOutput : false,
               passThreshold : 80,
               unstableThreshold: 20.0,
               onlyCritical : true,
               otherFiles : "*.png"
           ])
           echo "R ${currentBuild.result}"
           archiveArtifacts artifacts: 'artifacts/*', fingerprint: true
       }
   
   ```

   

###### 实践通过Jenkinsfile实现demo项目的验收测试

python-demo项目添加robot.txt文件：

`jenkins/pipelines/p10.yaml`

```powershell
pipeline {
    agent { label 'jnlp-slave'}
    
    options {
		buildDiscarder(logRotator(numToKeepStr: '10'))
		disableConcurrentBuilds()
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}

    environment {
        IMAGE_REPO = "172.21.51.67:5000/myblog"
        DINGTALK_CREDS = credentials('dingTalk')
        TAB_STR = "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
    }

    stages {
        stage('git-log') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        }        
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS = env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('CI'){
            failFast true
            parallel {
                stage('Unit Test') {
                    steps {
                        echo "Unit Test Stage Skip..."
                    }
                }
                stage('Code Scan') {
                    steps {
                        container('tools') {
                            withSonarQubeEnv('sonarqube') {
                                sh 'sonar-scanner -X'
                                sleep 3
                            }
                            script {
                                timeout(1) {
                                    def qg = waitForQualityGate('sonarqube')
                                    if (qg.status != 'OK') {
                                        error "未通过Sonarqube的代码质量阈检查，请及时修改！failure: ${qg.status}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('build-image') {
            steps {
                container('tools') {
                    retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('push-image') {
            steps {
                container('tools') {
                    retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('deploy') {
            steps {
                container('tools') {
                    sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                    timeout(time: 1, unit: 'MINUTES') {
                        sh "kubectl apply -f deploy/;sleep 20;"
                    }
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('Accept Test') {
            steps {
                    container('tools') {
                        sh 'robot -i critical  -d artifacts/ robot.txt|| echo ok'
                        echo "R ${currentBuild.result}"
                        step([
                            $class : 'RobotPublisher',
                            outputPath: 'artifacts/',
                            outputFileName : "output.xml",
                            disableArchiveOutput : false,
                            passThreshold : 80,
                            unstableThreshold: 20.0,
                            onlyCritical : true,
                            otherFiles : "*.png"
                        ])
                        echo "R ${currentBuild.result}"
                        archiveArtifacts artifacts: 'artifacts/*', fingerprint: true
                    }
            }
        }
    }
    post {
        success { 
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """ 
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😖❌ 构建失败 ❌😖  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}  \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}
```

在Jenkins中查看robot的构建结果。

> 



#### 小结

思路：

1. 讲解最基础的Jenkins的使用
2. Pipeline流水线的使用
3. Jenkinsfile的使用
4. 多分支流水线的使用
5. 与Kubernetes集成，动态jnlp slave pod的使用
6. 与sonarqube集成，实现代码扫描
7. 与Robotframework集成，实现验收测试



问题：

1. Jenkinsfile过于冗长
2. 多个项目配置Jenkinsfile，存在很多重复内容
3. 没有实现根据不同分支来部署到不同的环境
4. Java项目的构建
5. k8s部署后，采用等待的方式执行后续步骤，不合理