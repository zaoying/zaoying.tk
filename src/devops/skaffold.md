# 通过skaffold快速部署微服务

> 随着技术的不断发展，程序员们熟悉的传统单体应用开发流程，渐渐地无法适应当下微服务化的潮流趋势。同时随着云原生开发的理念不断推广，越来越多的服务运行在不可变的基础设施之上，随之而来的是传统单体应用开发流程与云化程度日益加深服务之间的隔阂越发巨大，开发人员越来越难以容忍重复繁琐且容易出错的低效率开发流程。因此，一款面向开发人员而运维实施人员的持续构建与持续部署工具 `skaffold` 应运而生

## skaffold简介

[skaffold]() 是一款 `Google` 推出的持续构建与持续部署工具，它主要面向开发人员而非运维实施人员，目标是打破本地开发与云化部署之间的隔阂，减轻开发人员的心智负担，帮助开发人员专注于出色地完成日常开发工作，避免开发人员在纷乱繁杂的运维流程中过多消耗宝贵的精力与时间。

### 基本架构

![architecture](./images/architecture.png)

`skaffold` 的工作流按照开发流程的不同阶段，分为4个部分组成：

1. 本地开发(文件同步)
2. 持续构建
3. 持续测试
4. 持续部署

以上四个部分均可以根据实际需求进行定制化修改。

#### 本地开发

`skaffold` 对主流的编程语言以及配套使用的技术栈都有着非常不错的支持，例如 `Go` 、`Java`、`JavaScript` 等

本地开发的核心内容是 `文件同步`，文件同步的监听对象大概可以分为 `源代码` 和 `编译产物` 。

`skaffold` 官方的推荐做法是监听源代码变动，然后自动化把源代码复制到Docker容器中进行编译和构建。

这种做法的问题不少，首先是源代码变动非常频繁，而编译和构建过程往往非常耗时，因此自动触发构建不太合理。

其次，在Docker容器中编译和构建，需要掌握编写 `Multi Stage Dockerfile` 技能，否则构建出来的镜像大小会占据非常大的空间，另外还要消耗本就不宽裕的带宽进行镜像传输。

最后，在Docker容器中编译和构建，要解决环境变量，代理设置、缓存构建中间结果等一系列问题，对新手非常不友好。

因此，个人推荐，在本地开发环节尽量采用手动触发编译构建，通过监听编译产物的方式来触发热更新等流程。

#### 持续构建

因为选择手动触发编译，所以本环节的内容主要讲述如何打包镜像的内容

目前 `skaffold` 官方支持的构建方式有三种：`Docker`、`Jib(maven/gradle)`、`Bazel`

* Docker
* Jib(maven/gradle)
* Bazel

这里以最常见 `Docker` 为例：

```yaml
build:
  local:
    push: false                        # 镜像打包成功后是否推送到远端的镜像仓库
  artifacts:                           # 支持打包多个不同组件的镜像
    - image: datacenter-eureka         # 打包后的镜像名称
      context: "eureka"                # Dockerfile相对路径，就放在eureka目录下
      docker:
        dockerfile: Dockerfile
    - image: datacenter-school         # 打包后的镜像名称
      context: "school"                # Dockerfile相对路径
      docker:
        dockerfile: Dockerfile
    - image: datacenter-teacher        # 打包后的镜像名称
      context: "teacher"               # Dockerfile相对路径
      docker:
        dockerfile: Dockerfile
    - image: datacenter-student        # 打包后的镜像名称
      context: "student"               # Dockerfile相对路径
      docker:
        dockerfile: Dockerfile
```

当运行 `skaffold dev` 时，会按照 `编译 —> 构建 -> 测试 -> 部署` 的标准流程走一遍。
当监听到指定路径下的文件发生变化时，skaffold工具会尝试通过类似于 `kubectl cp` 命令的方式，直接把产生变化后的文件拷贝到运行中的容器内部，避免重新走一遍编译构建/上传镜像的步骤，减少同步代码更改而消耗的时间。

需要特别注意，这种方式对于支持 `代码热更新` 的技术栈非常实用，例如 `Java` 和 `Javascript`，但对于 `Go` 这类不支持 `热更新` 的技术栈来说效果十分有限，因为即便文件同步完成，依然重启主进程才能让修改后的功能生效。

接着说 `Jib(maven/gradle)` ，`Jib` 也是由谷歌开发的一款专门针对 `Java` 生态的 `CI/CD` 工具，跟 `skaffold` 通用 `CI/CD` 不同，同时还有 `VSCode` 和 `IDEA` 插件， 但作者本人并没有用过，所以等以后有机会再展开讲。

至于 `Bazel` 是微软开发的全平台构建工具，主要支持 `C#` 语言，甚至连前端相关 `Javascript` 项目也可以使用，但缺点就是非常笨重，这里也不展开讲。

此外， skaffold 还支持 `Customize` 自定义构建，这个方式的构建更自由可控，比如有些 `WSL` 环境的用户不愿意为了安装 `Docker` 环境而反复折腾，甚至有些企业内部不允许员工在开发电脑安装虚拟机等等，通过自定义构建流程都可以解决，放到文章最后再详细讲解。

#### 持续测试

`skaffold` 官方的测试方案是把代码拷贝到定制化的测试环境容器中执行测试用例，这种方法非常麻烦，测试相关的内容这里就不展开讲。
感兴趣的读者可以查看 [skaffold官方配置文档](https://skaffold.dev/docs/references/yaml/)

#### 持续部署

`skaffold` 官方支持的部署方式有很多种，这里以 `helm` 为例：

```yaml
deploy:
  helm:
    releases:
    - name: datacenter
      chartPath: package
      artifactOverrides:
        image:
          eureka: datacenter-eureka      # 镜像名称要跟前面构建环节的镜像名称保持一致，但不能出现镜像标签
          school: datacenter-school      # 镜像名称要跟前面构建环节的镜像名称保持一致，但不能出现镜像标签
          teacher: datacenter-teacher    # 镜像名称要跟前面构建环节的镜像名称保持一致，但不能出现镜像标签
          student: datacenter-student    # 镜像名称要跟前面构建环节的镜像名称保持一致，但不能出现镜像标签
      imageStrategy:
        helm: {}
```

### 配置参考

完整的配置文件可以参考：[datacenter](https://github.com/zaoying/datacenter/blob/master/skaffold.yaml)

## 上手实践

### 前期准备

* [skaffold](https://github.com/GoogleContainerTools/skaffold/releases/latest)
* [docker cli](https://github.com/docker/cli/releases/latest)
* [helm](https://github.com/helm/helm/releases/latest)
* [kubectl](https://dl.k8s.io/release/v1.23.0/bin/windows/amd64/kubectl.exe)

一共需要安装以上4个组件，如果你电脑已安装 [chocolatey](https://chocolatey.org/install) 等包管理器，直接运行以下命令即可一键安装

```sh
choco install -y skaffold docker-cli kubernetes-helm kubernetes-cli
```

如果不愿意安装 `chocolatey` ，也点击上述四个组件的链接，下载到本地，再讲二进制加入 `PATH` 环境变量，详细安装过程不再赘述。

### 基本开发环境配置

像 `JDK` 、`maven` 或 `Gradle` 这类Java开发必备的工具请自行安装，这里就不展开讲了。

### 初始化helm chart

在一个空白目录下运行 `helm create datacenter` 命令，即可快速创建 `chart` 包，包的目录结构如下所示：

```txt
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```
可以根据自身的实际需求，增删修改包的文件内容，例如这里用不到 `hpa.yaml` 、 `serviceaccount.yaml` 和 `tests/*` ，所以都删除了。

然后把 `datacenter` 重命名为 `package` ，然后移动到原本的代码目录下，这是约定成俗的习惯。

### 部署MySQL服务

经典的Web应用往往离不开数据库，而在k8s上运行数据库，则需要提供持久化存储，否则数据库的容器重启后数据就丢失了。

首先，在 `package/templates` 目录下创建 `pv.yaml` 文件，然后写入以下内容：

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: mysql-pv-volume
  namespace: spring-cloud
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/opt/data/mysql"
```
解释：创建持久化卷 `PersistentVolume` ，简称 `PV`，存储路径就用宿主机目录 `/opt/data/mysql`

然后，在同一个目录下创建 `pvc.yaml` 文件，然后写入以下内容：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  namespace: spring-cloud
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

解释：创建持久卷使用声明 `PersistentVolumeClaim` ，简称 `PVC`，绑定前面创建的 `PV`。  
`PVC` 的容量是 `2G` ，必须小或等于 `PV` 的容量，否则无法绑定，可以根据实际情况调整容量大小。

> 为了防止卸载过程中意外删除PV卷导致数据丢失的情况，helm不会执行删除 PV 的操作，必须要用户手动执行。
> 因此如果部署过程出现PVC与PV无法绑定而导致无法继续的情况，请手动删除再重新PV以及PVC的方式排除故障


最后，在同一个目录下创建 `statefulset.yaml` 文件，然后写入以下内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql                  # 同一个命名空间的其他服务可以通过域名 “mysql” 来访问 MySQL 服务
  namespace: spring-cloud      # 通过命名空间来隔离不同的项目
spec:
  type: ClusterIP
  ports:
  - name: mysql
    protocol: TCP
    port: 3306
    targetPort: 3306
  selector:
    app: mysql                 # 通过定义标签选择器，只转发请求给带有 app: mysql 的Pod
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: spring-cloud
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: bocloud@2019    # 密码属于高度敏感机密，建议在生成环境中通过 ServiceAccount 和 ConfigMap 等方式注入
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim   # 将前面定义的pvc挂载成卷，给容器使用
```
解释：创建 mysql 的 `Service` 和 `StatefulSet`。由于 mysql 是个数据库，属于 `有状态应用` ，所以建议使用 `StatefulSet` 来管理。
另外由于 `k8s` 的机制问题，Pod 重启后IP地址会改变，所以Pod之间的通信不适合直接通过访问 Pod IP 的方式进行，最佳实践是通过创建特定 `Service` ,请求方的 Pod 向特定 Service 发送请求，再由特定 Service 转发请求给被请求方的 Pod。 

### 部署微服务

在 `package/templates` 目录下清空 `deployment.yaml` 文件，然后写入以下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: datacenter-dep
  namespace: spring-cloud      # 同一个项目的命名空间一定要相同
  labels:
    app: datacenter            # 自定义标签
spec:
  replicas: 1                       # 副本数量
  selector:
    matchLabels:
      app: datacenter               # 通过统计有多少个带有app: datacenter的Pod来确定副本的数量
  template:
    metadata:
      labels:
        app: datacenter             # 给Pod打上app: datacenter标签，方便统计
    spec:
      containers:
      - name: school
        image: {{.Values.image.school.repository}}:{{.Values.image.school.tag}} # 注入真正的镜像
        imagePullPolicy: IfNotPresent
        env:
        - name: MYSQL_ADDRESS
          value: mysql:3306
        - name: MYSQL_PASSWORD
          value: bocloud@2019           # 密码属于高度敏感机密，不建议在真实环境使用明文密码，这里仅为展示
        ports:
        - containerPort: 8084
      - name: teacher
        image: {{.Values.image.teacher.repository}}:{{.Values.image.teacher.tag}} # 注入真正的镜像
        imagePullPolicy: IfNotPresent
        env:
        - name: MYSQL_ADDRESS
          value: mysql:3306
        - name: MYSQL_PASSWORD
          value: bocloud@2019           # 密码属于高度敏感机密，不建议在真实环境使用明文密码，这里仅为展示
        ports:
        - containerPort: 8082
      - name: student
        image: {{.Values.image.student.repository}}:{{.Values.image.student.tag}} # 注入真正的镜像
        imagePullPolicy: IfNotPresent
        env:
        - name: MYSQL_ADDRESS
          value: mysql:3306
        - name: MYSQL_PASSWORD
          value: bocloud@2019           # 密码属于高度敏感机密，不建议在真实环境使用明文密码，这里仅为展示
        ports:
        - containerPort: 8083

```
解释：根据 `k8s` 的规范要求，应该通过 `Deployment` 来部署 `无状态应用` 。

然后，在同一个目录下清空 `service.yaml` 文件，然后写入以下内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: datacenter
  namespace: spring-cloud
spec:
  type: ClusterIP
  selector:
    app: datacenter
  ports:
    - name: school
      protocol: TCP
      port: 8084
      targetPort: 8084
    - name: teacher
      protocol: TCP
      port: 8082
      targetPort: 8082
    - name: student
      protocol: TCP
      port: 8083
      targetPort: 8083

```

解释：创建 `Service` 暴露到集群内部，供集群内部的其他服务调用

最后，修改 `package/values.yaml` 文件，然后写入以下内容：

```yaml
image:
  school:
    repository: datacenter-school
    tag: latest
  teacher:
    repository: datacenter-teacher
    tag: latest
  student:
    repository: datacenter-student
    tag: latest
```
解释：helm 推荐通过 `values.yaml` 文件统一管理 `chart` 模板的变量。
`skaffold` 也是通过修改 `values.yaml` 注入不同的镜像名称，动态更新运行中容器镜像

### 配置docker和kubectl

#### 安装 `docker` 用于打包镜像。
对于 `WSL1` 或者 嫌弃在 `WSL2` 安装 `docker` 环境太麻烦的 windows 用户，以及不想在本地安装 docker 的 Mac 用户，
可以尝试安装 `redhat` 开源的 `buildah` 

```sh
apt install buildah
```

* kubectl

### 代码热更新

代码热更新在日常的开发过程非常实用，可以加快特性开发与功能验证的效率。

skaffold 实现代码热更新的关键步骤是在构建环节中，跳过 `mvn package` 打包环节，
直接将编译的中间产物，例如 `.class` 字节码文件同步到运行中的容器中，在不重启容器的前提下实现代码热更新。

理论上来说，skaffold 的代码热更新功能同时适用于 `Java` 和 `Javascript` 等技术栈。

代码热更新的重点在于如何编写 skaffold 的自定义构建 `Customized` 环节代码：

```yaml
# 待补充
```

对于 `WSL1` 或者 嫌弃在 `WSL2` 安装 `docker` 环境太麻烦的 windows 用户，
可以尝试使用，
甚至把本地的编译产物传输到远程具有 `Docker` 环境的 `Linux` 服务器上构建，
从而彻底摆脱在本地构建对 `docker` 环境的依赖。

### 总结

本次演示所使用的微服务项目是很多年前笔者为了学习 `Spring Cloud` 而编写的 `Demo` 。
时隔多年 `Spring Cloud` 已经不再推荐 `Eureka` 作为服务发现与注册中心。
同时 `k8s` 本身也支持将 `CoreDNS` 作为服务发现/注册的组件使用。
所以读者不必纠结 `Demo` 代码中的错误，因为本文的重点是 `skaffold` 的配置与使用。