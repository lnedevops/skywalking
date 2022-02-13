# helm部署skywalking和实现全链路追踪

原文链接：

https://blog.csdn.net/nangonghen/article/details/110290450

https://www.jianshu.com/p/3a5fc697f7c2

## 一、概述

### 1.1环境

- 操作系统：centos 7.9
- k8s：v1.20.1
- helm：helm3.5
- skywalking：v8.8.1
- es版本：7.7.1

### 1.2skywalking概述

#### 1.2.1skywalking是什么

SkyWalking是一个开源的APM系统，为云原生分布式系统提供监控、链路追踪、诊断能力，支持集成多种编程语言的应用（java、php、go、lua等），也能和服务网格进行集成。除了支持代码侵入方式的集成，一个主要亮点也支持零代码入侵的集成（零代码侵入是和具体的编程语言相关的），是利用java agent的特性在jvm级别修改了运行时的程序，因此程序员在代码编辑期间不需要修改业务代码也能达到埋点的效果。后端存储支持es、mysql、tidb等多种数据库。
![20201128193220663](C:\Users\18311\Desktop\20201128193220663.png)

#### 1.2.2skywalking的java代理的使用

1）方式一：命令行方式

> 这种方式建议在docker构建java项目镜像时将命令加入

```bash
java \
-javaagent:/root/skywalking/agent/skywalking-agent.jar \
-Dskywalking.agent.service_name=app1 \
-Dskywalking.collector.backend_service=localhost:11800 \
-jar myapp.jar
```

2）方式二：环境变量方式

> 建议在pod中env时定义好SW_AGENT_COLLECTOR_BACKEND_SERVICES，SW_AGENT_NAME和JAVA_OPTS，同时构建java项目镜像时将$JAVA_OPTS加入启动命令中，例如：` ENTRYPOINT ["/bin/bash","-c","java ${JVM_OPTS} ${JAVA_OPTS} -jar app.jar ${APP_OPTS}"]`

```bash
export SW_AGENT_COLLECTOR_BACKEND_SERVICES=10.0.0.1:11800,10.0.0.2:11800
export SW_AGENT_NAME=demo1
export JAVA_OPTS=-javaagent:/root/skywalking/agent/skywalking-agent.jar

java \
$JAVA_OPTS \
-jar myapp.jar
```

## 二、部署

### 2.1部署es集群

参考：http://www.mydlq.club/article/13/

### 2.2部署skywalking

**skywalking chart 的 GitHub地址为：https://github.com/apache/skywalking-kubernetes**

#### 2.2.1使用 [Artifact Hub](https://links.jianshu.com/go?to=https%3A%2F%2Fartifacthub.io%2F) 提供的 chart 部署 skywalking

添加 skywalking chart 仓库的命令如下：

```bash
helm repo add choerodon https://openchart.choerodon.com.cn/choerodon/c7n
```

使用下面的命令在仓库中搜索 skywalking：

```bash
helm search repo skywalking
NAME                        CHART VERSION   APP VERSION DESCRIPTION                 
choerodon/skywalking        6.6.0           6.6.0       Apache SkyWalking APM System
choerodon/skywalking-oap    0.1.3           0.1.3       skywalking-oap for Choerodon
choerodon/skywalking-ui     0.1.4           0.1.4       skywalking-ui for Choerodon 
choerodon/chart-test        1.0.0           1.0.0       skywalking-ui for Choerodon 
```

为了做一些自定义的配置，比如修改版本号等，使用下面的命令下载 chart 到本地：

```bash
# 下载 chart 到本地
[root@k8s-node01 chart-test]# helm pull choerodon/skywalking
# 查看下载的内容
[root@k8s-node01 chart-test]# ll
total 12
-rw-r--r-- 1 root root 10341 Apr 21 11:12 skywalking-8.8.1.tgz
# 解压 chart 包
[root@k8s-node01 chart-test]# tar -zxvf skywalking-8.8.1.tgz 
skywalking/Chart.yaml
skywalking/values.yaml
skywalking/templates/_helpers.tpl
skywalking/templates/istio-adapter/adapter.yaml
skywalking/templates/istio-adapter/handler.yaml
skywalking/templates/istio-adapter/instance.yaml
skywalking/templates/istio-adapter/rule.yaml
skywalking/templates/mysql-init.job.yaml
skywalking/templates/oap-clusterrole.yaml
skywalking/templates/oap-clusterrolebinding.yaml
skywalking/templates/oap-deployment.yaml
skywalking/templates/oap-role.yaml
skywalking/templates/oap-rolebinding.yaml
skywalking/templates/oap-serviceaccount.yaml
skywalking/templates/oap-svc.yaml
skywalking/templates/ui-deployment.yaml
skywalking/templates/ui-ingress.yaml
skywalking/templates/ui-svc.yaml
skywalking/.auto_devops.sh
skywalking/.choerodon/.docker/config.json
skywalking/.gitlab-ci.yml
skywalking/.helmignore
skywalking/Dockerfile
skywalking/README.md
```

自定义配置，可以修改 `values.yaml`文件：

```undefined
vim skywalking/values.yaml
```

可以看到，这里的 skywalking 默认使用的是 mysql 数据库，使用的 skywalking 的版本是 6.6.0，可以根据自己的需求修改对应的版本号，修改完成后保存退出。更多的配置修改说明可以查看 [skywalking 官方 chart 说明](https://links.jianshu.com/go?to=https%3A%2F%2Fartifacthub.io%2Fpackages%2Fhelm%2Fchoerodon%2Fskywalking)。

可以使用下面的命令安装 chart，并指定自定义的配置文件：

```bash
# Usage:  helm install [NAME] [CHART] [flags]
helm install  skywalking skywaling -f skywalking/values.yaml
```

使用上面的命令会将 skywalking 安装在 `default` 命名空间，我们可以使用 `-n namespace` 参数指定到特定的命名空间，前提是这个命名空间必须存在。

安装成功后可以使用下面的命令查看安装的 chart，安装后的 chart 叫做 release：

```cpp
helm list
```

可以使用下面的命令卸载 chart：

```csharp
# Usage:  helm uninstall RELEASE_NAME [...] [flags]
helm uninstall skywalking -n default
```

#### 2.2.2从 GitHub 下载 chart 源文件部署 skywalking

> skywalking chart 的 GitHub地址为：[https://github.com/apache/skywalking-kubernetes](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fapache%2Fskywalking-kubernetes)

使用下面的命令 clone 代码：

```php
git clone https://github.com/apache/skywalking-kubernetes
```

进到 skywalking 的 chart 目录，可以看到下面的内容：

```csharp
[root@k8s-node01 chart-test]# cd skywalking-kubernetes/chart/skywalking/
[root@k8s-node01 skywalking]# ll
total 84
-rw-r--r-- 1 root root  1382 Apr 21 11:35 Chart.yaml
drwxr-xr-x 3 root root  4096 Apr 21 11:35 files
-rw-r--r-- 1 root root   877 Apr 21 11:35 OWNERS
-rw-r--r-- 1 root root 42593 Apr 21 11:35 README.md
drwxr-xr-x 3 root root  4096 Apr 21 11:35 templates
-rw-r--r-- 1 root root  1030 Apr 21 11:35 values-es6.yaml
-rw-r--r-- 1 root root  1031 Apr 21 11:35 values-es7.yaml
-rw-r--r-- 1 root root  1366 Apr 21 11:35 values-my-es.yaml
-rw-r--r-- 1 root root 10184 Apr 21 11:35 values.yaml
```

可以看到作者非常贴心的为我们定义了三个自定义配置文件：`values-es6.yaml` 、`values-es7.yaml` 和 `values-my-es.yaml`，分别对应使用 es6、es7 和 外部 es 存储的配置。因为我这里使用的是外部自有的 es 集群，并且 es 的版本是 7.7.1，所以我需要修改 `values-my-es.yaml` 文件如下：



```yaml
# Default values for skywalking.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

oap:
  image:
    tag: 8.8.1
  storageType: elasticsearch

ui:
  image:
    tag: 8.8.1
  service:	#此处开启nodeport提供访问
    type: NodePort
    externalPort: 80
    internalPort: 8080
    nodePort: 30008

elasticsearch:
  enabled: false
  config:               # For users of an existing elasticsearch cluster,takes effect when `elasticsearch.enabled` is false
    host: elasticsearch-master-headless.es	# 我的是外部es集群，此处连接master节点的服务，而不是data和client节点的服务
    port:
      http: 9200
    user: "xxx"         # [optional]
    password: "xxx"     # [optional]

```

还可以修改 `values.yaml` 文件，比如开启 ingress，更多详细的配置可以查看 GitHub 中 [skywalking-kubernetes](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fapache%2Fskywalking-kubernetes) 的说明，根据需要配置好之后就可以使用下面的命令安装 chart：

```bash
helm install "${SKYWALKING_RELEASE_NAME}" skywalking -n "${SKYWALKING_RELEASE_NAMESPACE}"  -f ./skywalking/values-my-es.yaml
```

安装完成以后，可以通过下面的命令查看 pod 是否正常启动：

```kotlin
[root@k8s-node01 ~]# kubectl get pod -n default
NAME                                     READY   STATUS      RESTARTS   AGE
skywalking-es-init-v6sbn                 0/1     Completed   0          1h
skywalking-oap-5c4d5bf887-4cvjk          1/1     Running     0          1h
skywalking-oap-5c4d5bf887-g75fj          1/1     Running     0          1h
skywalking-ui-6cd4bbd858-sbpvt           1/1     Running     0          1h
```

## 三、使用sidecar将pod接入链路追踪

### 3.1概述

#### 3.1.1简介

前面简单介绍了使用 helm 部署 skywalking，下面介绍如何使用 sidecar 将 pod 接入链路追踪。Java微服务接入skywalking 可以使用 `SkyWalking Java Agent` 来上报监控数据，这就需要 java 微服务在启动参数中通过 `-javaagent:<skywalking-agent-path>` 指定 skywalking agent 探针包，通常有以下三种方式集成：

- 使用官方提供的基础镜像 `skywalking-base`；
- 将 agent 包构建到已存在的镜像中；
- 通过 sidecar 模式挂载 agent；

前面两种方式在前面的文章中有简单介绍，这里主要介绍如何使用 sidecar 将 pod 接入链路追踪，这种方式不需要修改原来的基础镜像，也不需要重新构建新的服务镜像，而是会以sidecar模式，通过共享的 volume 将 agent 所需的相关文件直接挂载到已经存在的服务镜像中。sidecar模式原理很简单，就是在 pod 中再部署一个初始容器，这个初始容器的作用就是将 skywalking agent 和 pod 中的应用容器共享。

#### 3.1.2 什么是初始化容器 init container

Init Container 就是用来做初始化工作的容器，可以是一个或者多个，如果有多个的话，这些容器会按定义的顺序依次执行，只有所有的 Init Container 执行完后，主容器才会被启动。我们知道一个Pod里面的所有容器是共享数据卷和网络命名空间的，所以 Init Container 里面产生的数据可以被主容器使用到的。

#### 3.1.3 自定义skywalking agent镜像

在开始以 sidecar 方式将一个 java 微服务接入 skywalking 之前，我们需要构建 skywalking agent 的公共镜像，具体步骤如下：

- 使用下面的命令下载 skywalking agent 并解压：

```ruby
# 下载 skywalking-8.5.0 for es6 版本的发布包，与部署的 skywalking 后端版本一致
wget https://www.apache.org/dyn/closer.cgi/skywalking/8.5.0/apache-skywalking-apm-8.5.0.tar.gz
# 将下载的发布包解压到当前目录
tar -zxvf apache-skywalking-apm-8.5.0.tar.gz
```

- 在前面步骤中解压的 skywalking 发行包的同级目录编写 Dockerfile 文件，具体内容如下：

```ruby
FROM busybox:latest
ENV LANG=C.UTF-8
RUN set -eux && mkdir -p /opt/skywalking/agent/
ADD apache-skywalking-apm-bin/agent/ /opt/skywalking/agent/
WORKDIR /
```

在上述 Dockefile 文件中使用的基础镜像是 bosybox 镜像，而不是 SkyWalking 的发行镜像，这样可以确保构建出来的sidecar镜像保持最小。

- 使用下面的命令构建镜像：

```css
docker build -t skywalking-agent-sidecar:8.5.0 .
```

### 3.2 sidecar模式接入skywalking

上面我们通过手工构建的方式构建了 SkyWalking Java Agent 的公共 Docker 镜像，接下来我们将演示如何通过编写 Kubernetes 服务发布文件，来将 Java 服务发布到 K8s 集群的过程中自动以 SideCar 的形式集成Agent 并接入 SkyWalking 服务。

1）微服务来自网上的若依：**https://gitee.com/y_project/RuoYi**，并做了一些修改。
2）我的业务服务部署在另外一个k8s集群中，因此skywalking agent访问的是位于另一个集群中的skywalking oap服务的NodePort。
3）每个yaml文件都可以直接使用，需要根据实际情况修改环境变量SW_AGENT_COLLECTOR_BACKEND_SERVICES。在我的例子中SW_AGENT_COLLECTOR_BACKEND_SERVICES=skywalking-oap.es.svc:11800。

## 四、部署服务

这里只展示一个服务的部署，其他服务同样如此

```yaml
apiVersion: v1
kind: Service
metadata:
  name: #APP_NAME
  labels:
    app: #APP_NAME
spec:
  type: ClusterIP
  ports:
    - name: server          #服务端口
      port: 8080
      targetPort: 8080
    - name: management      #监控及监控检查的端口
      port: 8081
      targetPort: 8081
  selector:
    app: #APP_NAME
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: #APP_NAME
  labels:
    app: #APP_NAME
spec:
  replicas: #APP_REPLICAS
  selector:
    matchLabels:
      app: #APP_NAME
  strategy:
    type: Recreate          #设置更新策略为删除策略
  template:
    metadata:
      labels:
        app: #APP_NAME
    spec:
      imagePullSecrets:
        - name: aliyun-registry-secret    #镜像拉取秘钥
      initContainers:
        - image: registry.cn-hangzhou.aliyuncs.com/fsl-devops/skywalking-agent-sidecar:8.5.0
          name: skywalking-sidecar
          command: ["sh"]
          args: ["-c","mkdir -p /opt/sw/agent && cp -rf /usr/skywalking/agent/* /opt/sw/agent"]
          volumeMounts:
            - name: sw-agent
              mountPath: /opt/sw/agent
      containers:
        - name: #APP_NAME
          image: #APP_IMAGE_NAME
          imagePullPolicy: Always
          ports:
            - containerPort: 8080   #服务端口
              name: server
            - containerPort: 8081   #监控及监控检查的端口
              name: management
          env:
            - name: "update_uuid"
              value: "#APP_UUID"    #生成的随机值，放置执行kubectl apply时能够执行
            - name: JAVA_OPTS
              value: "-javaagent:/opt/sw/agent/skywalking-agent.jar"
            - name: SW_AGENT_NAME
              value: "#APP_NAME"
            - name: SW_AGENT_COLLECTOR_BACKEND_SERVICES
              value: "skywalking-oap.es.svc:11800"
          volumeMounts:
            - mountPath: /opt/sw/agent
              name: sw-agent
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 1000m
              memory: 512Mi
          readinessProbe:
            httpGet:
              port: 8081      #需要提前开启springCloud项目的8081检测端口
              path: /actuator/health
            initialDelaySeconds: 120      #等待初始时间为120秒
            periodSeconds: 5
            timeoutSeconds: 10
          livenessProbe:
            httpGet:
              port: 8081
              path: /actuator/health
            initialDelaySeconds: 60
            periodSeconds: 5
            timeoutSeconds: 10
      volumes:
        - name: sw-agent
          emptyDir: {}
```


