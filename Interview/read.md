[TOC]

# 笔记

### Vue 中diff算法

> Vue借鉴了react的diff算法，只会同级比较，不会跨级比较

> 在新老dom的首位和末位节点上，分别建立索引，如老的dom的第一个节点索引标注为oldStart(os)，最后一个节点oldEnd(oe)，新节点的第一个节点索引标注为newStart(ns), 最后一个节点薇newEnd(ne)

> 新老两组节点相互比较，最多需要比较4次，即os->ns, os->ne, oe->ns, oe->ne

> 4次比较后，产生5种结果

> os==ns,即旧dom的开始节点，与新dom的开始节点相等，保持os节点位置不变，os,ns的索引向后移动一位

> os==ne,即旧dom的开始节点，与新dom的结束节点相等，将os节点移动到oe的索引位置后面，os索引向后移动一位，ne索引向前移动一位

> oe==ns,即旧dom的结束节点，与新dom的开始节点相等，将oe节点移动到os的索引位置前，oe索引向前移动一位，ns索引向后移动一位

> oe==ne,即旧的dom结束节点，与新dom的结束节点相等，保持oe节点的位置不变，oe,ne的索引向前移动一位

> 两组节点互不相等，在os与oe间，查找是否有ns相同的节点，如果有，则移动到os节点位置前，如果没有，则创建ns节点，并插入到os节点前，ns索引向后移动一位

> 如果os>oe，表示olddom先遍历完，就创建ns,ne之间的节点，插入到dom的ne前，结束比较

> 如果ns>ne，表示newdom先遍历完，说明os,oe之间的节点是多余的，直接删除，结束比较

> 直到出现任意一方的start>end，则有一方遍历结束，整个比较也结束

![MPG线程模型](../../Public/images/Ciqc1F8X5ymAf7NvAAE_PDbjFtw120.png)

- M，即 machine，相当于内核线程在 Go 进程中的映射，它与内核线程一一对应，代表真正执行计算的资源。在 M 的生命周期内，它只会与一个内核线程关联。

- P，即 processor，代表 Go 代码片段执行所需的上下文环境。M 和 P 的结合能够为 G 提供有效的运行环境，它们之间的结合关系不是固定的。P 的最大数量决定了 Go 程序的并发规模，由 runtime.GOMAXPROCS 变量决定。

- G，即 goroutine，是一种轻量级的用户线程，是对代码片段的封装，拥有执行时的栈、状态和代码片段等信息。

> 在实际执行过程中，M 和 P 共同为 G 提供有效的运行环境（如下图），多个可执行的 G 顺序挂载在 P 的可执行 G 队列下面，等待调度和执行。当 G 中存在一些 I/O 系统调用阻塞了 M 时，P 将会断开与 M 的联系，从调度器空闲 M 队列中获取一个 M 或者创建一个新的 M 组合执行， 保证 P 中可执行 G 队列中其他 G 得到执行，且由于程序中并行执行的 M 数量没变，保证了程序 CPU 的高利用率。

![M和P结合示意图](../../Public/images/Ciqc1F8X5zuAQrDLAAEpRhFm8n4546.png)

> 当 G 中系统调用执行结束返回时，M 会为 G 捕获一个 P 上下文，如果捕获失败，就把 G 放到全局可执行 G 队列等待其他 P 的获取。新创建的 G 会被放置到全局可执行 G 队列中，等待调度器分发到合适的 P 的可执行 G 队列中。M 和 P 结合后，会从 P 的可执行 G 队列中无锁获取 G 执行。当 P 的可执行 G 队列为空时，P 才会加锁从全局可执行 G 队列获取 G。当全局可执行 G 队列中也没有 G 时，P 会尝试从其他 P 的可执行 G 队列中“剽窃” G 执行。

#### goroutine 和 channel

> 并发程序中的多个线程同时在 CPU 执行，由于资源之间的相互依赖和竞态条件，需要一定的并发模型协作不同线程之间的任务执行。Go 中倡导使用 CSP 并发模型来控制线程之间的任务协作，CSP 倡导使用通信的方式来进行线程之间的内存共享。

> Go 是通过 goroutine 和 channel 来实现 CSP 并发模型的：

- goroutine，即协程，Go 中的并发实体，是一种轻量级的用户线程，是消息的发送和接收方；

- channel，即通道， goroutine 使用通道发送和接收消息。

```go
channel <- val // 发送消息
val := <- channel // 接收消息
val, ok := <- channel // 非阻塞接收消息
```

> 使用 for:range 从 channel 中循环接收消息，只有当相应的 channel 被内置函数 close 后，该循环才会结束。channel 在关闭之后不可以再用于发送消息，但是可以继续用于接收消息，从关闭的 channel 中接收消息或者正在被阻塞的 goroutine 将会接收零值并返回。还有一个需要注意的点是，main 函数由主 goroutine 启动，当主 goroutine 即 main 函数执行结束，整个 Go 程序也会直接执行结束，无论是否存在其他未执行完的 goroutine。

##### 1. select 多路复用

> select 的形式与 switch 类似，但是要求 case 语句后面必须为 channel 的收发操作

##### 2. Context 上下文

> 当需要在多个 goroutine 中传递上下文信息时，可以使用 Context 实现。Context 除了用来传递上下文信息，还可以用于传递终结执行子任务的相关信号，中止多个执行子任务的 goroutine。Context 中提供以下接口：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

- Deadline 方法，返回 Context 被取消的时间，也就是完成工作的截止日期；

- Done，返回一个 channel，这个 channel 会在当前工作完成或者上下文被取消之后关闭，多次调用 Done 方法会返回同一个 channel；

- Err 方法，返回 Context 结束的原因，它只会在 Done 返回的 channel 被关闭时才会返回非空的值，如果 Context 被取消，会返回 Canceled 错误；如果 Context 超时，会返回 DeadlineExceeded 错误。

- Value 方法，可用于从 Context 中获取传递的键值信息。

> 在 Web 请求的处理过程中，一个请求可能启动多个 goroutine 协同工作，这些 goroutine 之间可能需要共享请求的信息，且当请求被取消或者执行超时时，该请求对应的所有 goroutine 都需要快速结束，释放资源。Context 就是为了解决上述场景而开发的。

### 08 | 如何基于 Go-kit 开发 Web 应用：从接口层到业务层再到数据层

#### 使用 Go Modules 管理项目依赖

> go.mod 文件生成之后，会被 go toolchain 掌控维护，在我们执行 go run、go build、go get、go mod 等各类命令时自动修改和维护 go.mod 文件中的依赖内容。

> 除了 go mod init，还有 go mod download 和 go mod tidy 两个 Go Modules 常用命令。其中，go mod download 命令可以在我们手动修改 go.mod 文件后，手动更新项目的依赖关系；go mod tidy 与 go mod download 命令类似，但不同的是它会移除掉 go.mod 中没被使用的 require 模块。

> Go-kit 推荐使用 transport、endpoint 和 service 3 层结构来组织项目，它们的作用分别为：

- transport 层，指定项目提供服务的方式，比如 HTTP 或者 gRPC 等 。

- endpoint 层，负责接收请求并返回响应。对于每一个服务接口，endpoint 层都使用一个抽象的 Endpoint 来表示 ，我们可以为每一个 Endpoint 装饰 Go-kit 提供的附加功能，如日志记录、限流、熔断等。

- service 层，提供具体的业务实现接口，endpoint 层中的 Endpoint 通过调用 service 层的接口方法处理请求。

### 09 | 案例：货运平台应用的微服务划分

> DDD 是以领域为核心的，实践 DDD 时要首先根据问题域划分出相关的领域，描述应用需要解决什么样的问题。领域中存在限界上下文，它用于解决领域内的特定问题，具备特定的职责，并存在边界。限界上下文内的领域模型具备较高的内聚性，不同边界之间需要透过显示边界进行通信。

![管理端领域和限界上下文示意图](../../Public/images/CgqCHl8hKWmAFEEaAAL1_BQzatI363.png)

> 领域对象能够表达出业务意图，它们具备数据和行为，包括实体、值对象和聚合等概念：

- 实体是一种对象，有唯一的标识识别自身，具备一定的生命周期，并在生命周期内根据状态提供不同的行为，一般需要考虑实体的持久化问题。

- 值对象没有唯一的标识，在领域模型中是不可变和可共享的。

- 聚合由一系列实体和值对象内聚而成，用以表达一个完整的领域概念。每个聚合都存在一个聚合根（根实体）。

![货运上下文数据项模型图](../../Public/images/CgqCHl8hMnGAMGm2AAEc4JPei6k235.png)

> 多个限界上下文需要进行集成，才能完整解决领域内的问题。考虑到限界上下文的各种概念存在边界隔离的问题，在集成时需要注意不同限界上下文中领域概念的映射和翻译，避免对我们已经设计好的领域模型造成污染。限界上下文主要有以下几种映射方式。

- 合作关系（Partnership）：两个上下文紧密合作的关系，一荣俱荣，一损俱损。

- 共享内核（Shared Kernel）：两个上下文依赖部分共享的模型。

> 一般来讲，我们会把领域对象的行为和数据都放在一起，以达到设计上高内聚和低耦合的目的，并屏蔽领域对象的细节，暴露公开的行为方法。当存在一些不归属于任何领域对象的领域行为或操作时，我们就将它们放在领域服务中。

> 领域服务是领域模型的一部分，应该尽量避免将过多的领域行为放在领域服务中，因为这会造成领域对象变成“贫血对象”，而一个合格的领域对象应该是行为和数据都兼并的。在我们微服务实践项目中，考虑到服务暴露粒度的问题，我们会把上下文暴露的领域行为都放到领域服务中，包括各个上下文的聚合根对外暴露的领域行为。

- 防腐层（Anticorruption Layer）：一个上下文通过一些适配和转换与另一个上下文交互。
- 客户方-供应方开发（Customer-Supplier Development）：上下文之间有组织的上下游依赖。
- 开放主机服务（Open Host Service）：定义一种协议来让其他上下文对本上下文进行访问。
- 遵奉者（Conformist）：下游上下文只能盲目依赖上游上下文。
- 发布语言（Published Language）：通常与 OHS 一起使用，用于定义开放主机的协议。
- 大泥球（Big Ball of Mud）：混杂在一起的上下文关系，边界不清晰。
- 另谋他路（Separate Way）：两个完全没有任何联系的上下文。

### 10 | 案例：微服务 Docker 容器化部署和 Kubernetes 容器编排

#### Docker 简介

> Docker 中存在 3 个非常重要的核心概念：

- 镜像（Image）
- 容器（Container）
- 仓库（Repository）

#### Docker 部署 user 服务

```bash
docker search redis 
docker pull redis:5.0 
# 查看本地的镜像列表
docker images redis 
```

```bash
# 如果之前运行过 此容器名的容器 则提示名称被占用
# docker run -itd --name 容器名 -p 宿主机端口:容器端口
docker run -itd --name redis-5.0 -p 6379:6379 redis:5.0
```

- `-i` 选项让容器的标准输入保持打开，
- `-t` 选项让 Docker 分配一个伪终端并绑定到容器的标准输入上，这两个选项一般配合使用；
- `-d` 选项指定容器以后台方式运行，启动后返回容器的 ID；
- `--name` 指令用以指定容器名，要求每个容器都不一样；
- `-p` 指令将容器的 6379 端口映射到宿主机的 6379 端口，在外部我们可以直接通过宿主机6379 端口访问 Redis。

```bash
# 查看容器的运行情况
docker ps 
```

```bash
# 在容器中执行命令
docker exec -it redis-5.0 /bin/bash
```

```bash
# 启动容器 该容器需要创建
docker start redis-5.0
# 停止容器
docker stop redis-5.0
# 杀死容器
docker kill redis-5.0
# 连接运行中容器
docker attach redis-5.0
# 移除容器
docker rm redis-5.0
```

> Dockerfile

```Dockerfile
FROM golang:latest 
WORKDIR /root/micro-go-course/section10/user 
COPY / /root/micro-go-course/section10/user 
RUN go env -w GOPROXY=https://goproxy.cn,direct 
RUN go build -o user 
ENTRYPOINT ["./user"] 
```

> 在上面的 Dockerfile 中出现了五种指令。

- From：Dockerfile 中必须出现的第一个指令，用于指定基础镜像，在上述例子中我们指定了基础镜像为 golang:latest 版本。
- WORKDIR：指定工作目录，之后的命令将会在该目录下执行。
- COPY：将本地文件添加到容器指定位置中。
- RUN：创建镜像执行的命令，一个 Dockerfile 可以有多个 RUN 命令。在上述 RUN 指令中我们指定了 Go 的代理，并通过 go build 构建了 user 服务。
- ENTRYPOINT：容器被启动后执行的命令，每个 Dockerfile 只有一个。我们通过该命令在容器启动后，又启动了 user 服务。

```bash
# 使用 Dockerfile 创建镜像
docker build -t user . 
```

> 其中，`-t` 选项用于指定镜像的名称和标签，不指定标签默认为 latest；`.` 指构建上下文

- [关于docker build 命令中 path 参数的介绍](https://gist.github.com/zhaokuohaha/4955a0a5d997a818c1c08bdb30c771f1)

```Dockerfile
FROM mysql:5.7 
WORKDIR /docker-entrypoint-initdb.d 
ENV LANG=C.UTF-8 
COPY init.sql . 
```

```bash
docker build -t mysql-for-user  .
docker run  -itd --name mysql-for-user -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql-for-user 
```

```bash
docker exec -it user /bin/bash 
```

#### Kubernetes 简介

> Kubernetes 主要有由两类节点组成：Master 节点主要负责管理和控制，是 Kubernetes 的调度中心；Node 节点受 Master 节点管理，属于工作节点，负责运行具体的容器应用。整体结构图如下所示：

![Kubernetes 集群架构图](../../Public/images/CgqCHl8j-j-AL1E1AADTHvJdwEQ723.png)

> Master 节点主要由以下几部分组成：

- API Server，对外提供 Kubernetes 的服务接口，供各类客户端使用；
- Scheduler，负责对集群内部的资源进行调度，按照预设的策略将 Pod 调度到相应的 Node 节点；
- Controller Manager，作为管理控制器，负责维护整个集群的状态；
- etcd，保存整个集群的状态数据。

> Node 节点的主要组成部分为：

- Pod，Kubernetes 创建和部署的基本操作单位，它代表了集群中运行的一个进程，内部由一个或者多个共享资源的容器组成，我们可以简单将 Pod 理解成一台虚拟主机，主机内的容器共享网络、存储等资源；
- Docker，是 Pod 中最常见的容器 runtime，Pod 也支持其他容器 runtime；
- Kubelet，负责维护调度到它所在 Node 节点的 Pod 的生命周期，包括创建、修改、删除和监控等；
- Kube-proxy，负责为 Pod 提供代理，为 Service 提供集群内部的服务发现和负载均衡，Service 可以看作一组提供相同服务的 Pod 的对外访问接口。

#### Kubernetes 部署 user 服务

```yaml
apiVersion: v1 
kind: Pod 
metadata: 
  name: user-service 
  labels: 
    name: user-service 
spec: 
  containers:                    #定义user容器，开放10086端口 
    - name: user 
      image: user 
      ports: 
        - containerPort: 10086 
      imagePullPolicy: IfNotPresent 
    - name: mysql                     #定义MySQL容器，开放3306端口 
      image: mysql-for-user 
      ports: 
        - containerPort: 3306 
      env: 
        - name: MYSQL_ROOT_PASSWORD 
          value: "123456" 
      imagePullPolicy: IfNotPresent 
    - name: redis                     #定义Redis容器，开放6379端口 
      image: redis:5.0 
      ports: 
        - containerPort: 6379 
      imagePullPolicy: IfNotPresent 
```

```bash
kubectl create -f user-service.yaml 
```

```bash
kubectl get pod user-service  
kubectl exec -ti user-service -n default  -- /bin/bash 
```

> 单个 Pod 不具备自我恢复的能力，当 Pod 所在的 Node 出现问题，Pod 就很可能被删除，这就会导致 Pod 中容器提供的服务被终止。为了避免这种情况的发生，可以使用 Controller 来管理 Pod，Controller 提供创建和管理多个 Pod 的能力，从而使得被管理的 Pod 具备自愈和更新的能力。常见的 Controller 有以下几种：

- Replication Controller，确保用户定义的 Pod 副本数保持不变；
- ReplicaSet，是 RC 的升级版，在选择器（Selector）的支持上优于 RC，RC 只支持基于等式的选择器，但 RS 还支持基于集合的选择器；
- Deployment，在 RS 的基础上提供了 Pod 的更新能力，在 Deployment 配置文件中 Pod template 发生变化时，它能将现在集群的状态逐步更新成 Deployment 中定义的目标状态；
- StatefulSets，其中的 Pod 是有序部署和具备稳定的标识，是一组存在状态的 Pod 副本。

```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata: 
  name: user-service 
  labels: 
    name: user-service 
spec: 
  replicas: 3 
  selector: 
      matchLabels: 
        name: user-service 
  template: 
    metadata: 
      labels: 
        name: user-service 
    spec: 
      containers:                    #定义user容器，开放10086端口 
        - name: user 
          image: user 
          ports: 
            - containerPort: 10086 
          imagePullPolicy: IfNotPresent 
        - name: mysql                     #定义MySQL容器，开放3306端口 
          image: mysql-for-user 
          ports: 
            - containerPort: 3306 
          env: 
            - name: MYSQL_ROOT_PASSWORD 
              value: "123456" 
          imagePullPolicy: IfNotPresent 
        - name: redis                     #定义Redis容器，开放6379端口 
          image: redis:5.0 
          ports: 
            - containerPort: 6379 
          imagePullPolicy: IfNotPresent 
```

```bash
kubectl create -f user-service-deployment.yaml 
```

```bash
kubectl get Deployment user-service 
```

> Deployment Controller 默认使用 RollingUpdate 策略更新 Pod，也就是滚动更新的方式；另一种更新策略是 Recreate，创建出新的 Pod 之前会先杀掉所有已存在的 Pod，可以通过 spec.strategy.type 标签指定更新策略。Deployment 的 rollout 当且仅当 Deployment 的 Pod template 中的 label 更新或者镜像更改时被触发，比如我们希望更新 redis 的版本：

```bash
kubectl set image deployment/user-service redis=redis:6.0 
```

> 这将触发 user-service Pod 的重新更新部署。当 Pod 被 Deployment Controller 管理时，单独使用 kubectl delete pod 无法删除相关 Pod，Deployment Controller 会维持 Pod 副本数量不变，这时则需要通过 kubectl delete Deployment 删除相关 Deployment 配置，比如删除 user-service 的 Deployment 配置，如下命令所示：

```bash
kubectl delete Deployment user-service 
```

### 11 | 案例：如何结合 Jenkins 完成持续化集成和自动化测试？

#### 持续集成与 Jenkins Pipeline

> Pipeline 是一套运行在 Jenkins 上的工作框架。它能够将多个节点中的任务连接起来，实现单个节点难以完成的复杂流程的编排和可视化工作。Pipeline 以代码的形式实现，它将一个流水线划分为多个 Stage，每个 Stage 代表了一组操作，比如构建、测试、部署等；而 Stage 内部又由多个 Step 组成，每一个 Step 就是基本的操作命令，比如打印日志 "echo" 等命令。

#### Go 的单元测试

> 在测试文件中主要存在以下三种函数类型：

1. 以 Test 作为函数名前缀的测试函数，一般用作单元测试，测试函数的逻辑行为是否正确；

2. 以 Benchmark 作为函数名前缀的基准测试函数，一般用来衡量函数的性能；

3. 以 Example 作为函数名前缀的示例函数，主要用于提供示例文档。

#### 使用 Pipeline 构建部署服务

> 在部署 Pipeline 服务之前，我们首先将 user 服务依赖的 MySQL 和 Redis 独立部署到 Kubernetes 上，这里我们以 Redis 的 yaml 配置为例：

```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata: 
  name: user-redis 
  labels: 
    name: user-redis 
spec: 
  replicas: 1 
  strategy: 
    type: RollingUpdate 
  selector: 
      matchLabels: 
        name: user-redis 
  template: 
    metadata: 
      labels: 
        name: user-redis 
    spec: 
      containers:                    #定义Redis容器，开放6379端口 
        - name: user-redis 
          image: redis:5.0 
          ports: 
            - containerPort: 6379 
          imagePullPolicy: IfNotPresent 
```

> user-redis.yaml 文件通过 Deployment Controller 管理 Pod，当 Controller 中的 Pod 出现异常被重启时，很可能导致 Pod 的 IP 发生变化。如果此时 user 服务通过固定 IP 的方式访问 Redis，很可能会访问失败。为了避免这种情况，我们可以为 user-redis Pod 定义一个 Service，配置文件描述如下：

```yaml
apiVersion: v1  
kind: Service 
metadata:  
  name: user-redis-service 
spec: 
  selector:  
    name: user-redis 
  ports:   
  - protocol: TCP 
    port: 6379 
    targetPort: 6379 
    name: user-redis-tcp 
```

> 在创建好 Pod 后，再执行 kubectl create -f user-redis-service.yaml 命令，即可为 user-redis Pod 生成一个 Service。Service 定义了一组 Pod 的逻辑集合和一个用于访问它们的策略，Kubernetes 集群会为 Service 分配一个固定的 Cluster IP，用于集群内部的访问。我们可以通过以下命令查看 Service 的信息，包括 Cluster IP 等信息：

```bash
kubectl get services 
```

> 通过 Cluster IP 访问 MySQL 和 Redis 等服务，我们就无须担心 Pod IP 的变化。

> 通过 Pipeline 部署服务到 Kubernetes 集群，主要有以下步骤：

1. 从 GitHub 中拉取代码；
2. 构建 Docker 镜像；
3. 上传 Docker 镜像到 Docker Hub；
4. 将应用部署 Kubernetes；
5. 接口测试。

> Pipeline 脚本是由 Groovy 语言实现，支持 Declarative（声明式）和 Scripted（脚本式）语法，我们接下来的演示就基于脚本式语法进行介绍。

```groovy
node {

    script {
        mysql_addr = '127.0.0.1' // service cluster ip
        redis_addr = '127.0.0.1' // service cluster ip
        user_addr = '127.0.0.1:30036' // nodeIp : port
    }

    // 使用 Jenkinsfile 会关联 Git 仓库，代码已经一起拉下来
    stage('get commit_id from github') {
        echo "first stage: get commit_id"
        script {
            commit_id = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
        }
    }

    stage('build image') {
        echo "second stage: build docker image"
        sh "docker build -t xingxingso/mgc-user:${commit_id} /"
    }

    stage('push image') {
        echo "third stage: push docker image to registry"
        sh "docker login -u kantchan -p xxxxxx" // user your docker account
        sh "docker push xingxingso/user:${commit_id}"
    }

    stage('deploy to Kubernetes') {
        echo "forth stage: deploy to Kubernetes"
        sh "sed -i 's/<COMMIT_ID_TAG>/${commit_id}/' user-service-11.yaml"
        sh "sed -i 's/<MYSQL_ADDR_TAG>/${mysql_addr}/' user-service-11.yaml"
        sh "sed -i 's/<REDIS_ADDR_TAG>/${redis_addr}/' user-service-11.yaml"
        sh "kubectl apply -f user.yaml"
    }

    stage('http test') {
        echo "fifth stage: http test"
        sh "cd transport && go test  -args ${user_addr}"
    }

}
```

> 使用 kubectl 将 user 服务部署到 Kubernetes 中。为了保证部署到正确版本的镜像，我们需要将 commit_id 替换到 user.yaml 文件中，以及将 mysqlAddr 和 redisAddr 作为环境变量输入，user.yaml 的配置如下：

```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata: 
  name: user-service 
  labels: 
    name: user-service 
spec: 
  replicas: 1 
  strategy: 
    type: RollingUpdate 
  selector: 
    matchLabels: 
      name: user-service 
  template: 
    metadata: 
      labels: 
        name: user-service 
    spec: 
      containers:                    #定义User容器，开放10086端口 
        - name: user 
          image: aoho/user:<COMMIT_ID_TAG> 
          ports: 
            - containerPort: 10086 
          imagePullPolicy: IfNotPresent 
          env: 
            - name: mysqlAddr 
              value: <MYSQL_ADDR_TAG> 
            - name: redisAddr 
              value: <REDIS_ADDR_TAG> 
```

### 12 | 服务注册与发现如何满足服务治理？

> 由于服务实例是动态部署的，每个服务实例的地址和服务信息都可能动态变化，这就势必需要一个中心化的组件对各个服务实例的信息进行管理，该组件管理了各个部署好的服务实例元数据，包括服务名、IP 地址、端口号、服务描述和服务状态等。

#### 什么是服务注册与发现

> 服务注册与发现由两部分组成：①服务注册，指服务实例在启动时将自身信息注册到服务注册与发现中心，并在运行时通过心跳等方式向服务注册与发现中心汇报自身服务状态；②服务发现，指服务实例根据服务名向服务注册与发现中心请求其他服务实例信息，用于进行接下来的远程调用。

> CAP 原理就是描述分布式系统下节点数据同步的基本定理。该原理由加州大学的 Eric Brewer 教授提出，分别指 Consistency（一致性）、Availability（可用性） 和 Partition tolerance（分区容忍性）。其具体含义如下：

- Consistency，指数据一致性，表示一个系统的数据信息（包括备份数据）在同一时刻都是一致的。在分布式系统下，同一份数据可能存在于多个不同的实例中，在数据强一致性的要求下，对其中一份数据的修改必须同步到它的所有备份中。在数据同步的任何时候，都需要保证所有对该份数据的请求将返回同样的状态。

- Availability，指服务可用性，要求服务在接收到客户端请求后，都能够给出响应。服务可用性考量的是系统的可用性，要求系统在高并发情况下和部分节点宕机的情况下，系统整体依然能够响应客户端的请求。

- Partition tolerance，指分区容忍性。在分布式系统中，不同节点之间是通过网络进行通信。基于网络的不可靠性，位于不同网络分区的服务节点可能会通信失败，如果系统能够容忍这种情况，说明它是满足分区容忍性的。如果系统不能够满足分区容忍性，那么将会限制分布式系统的扩展性，即服务节点的部署数量和地区都会受限，这就违背了分布式系统设计的初衷，所以一般来讲分布式系统都会满足 P，也就是分区容忍性。

#### 如何选择服务注册与发现框架

1. Consul
2. Etcd
3. ZooKeeper

![组件对比](../../Public/images/Ciqc1F8tPxCAT_4RAADBzRFlUA0352.png)

### 13 | 案例：如何基于 Consul 给微服务添加服务注册与发现？

#### Consul 集群

> Consul 集群中存在 Server 和 Client 两种角色节点，Server 中保存了整个集群的数据，而 Client 负责对本地的服务进行健康检查和转发请求到 Server 中，并且也保存有注册到本节点的服务实例数据。

> 关于 Server 节点，一般建议你部署 3 个或者 5 个节点，但并不是越多越好，因为这会增加数据同步的成本。Server 节点之间存在一个 Leader 和多个 Follower，通过 Raft 协议维护 Server 之间数据的强一致性。一个典型的 Consul 集群和微服务的部署方式如下：

![Consul 集群和微服务部署图](../../Public/images/Ciqc1F8zngqAbS-SAAF6iFt7jEw242.png)

#### 注册服务到 Consul

> Consul 提供 HTTP 和 DNS 两种方式访问服务注册与发现接口

> 由于我们是通过 Consul Client 进行服务注册与发现，所以接下来我们会首先介绍 Consul Client 中提供的用于服务注册、服务注销和服务发现的 HTTP API，如下所示：

```
/v1/agent/service/register // 服务注册接口 
/v1/agent/service/deregister/${instanceId} // 服务注销接口 
/v1/health/service/${serviceName} // 服务发现接口 
```

> 由于 Consul Client 部署在每一个 Node 节点中，我们可以直接获取 spec.nodeName（即 Pod 所在 Node 节点的主机名）作为 Consul Client 的地址传递给 Go 微服务，而 Go 微服务的 IP 地址即其所在 Pod 的 IP。在 Kubernetes 中启动该配置后即可在 Consul UI 中查看到该服务实例注册到 Consul 中

### 14 | 案例：如何在 Go-kit 和 Service Mesh 中进行服务注册与发现？

#### 使用 Go-kit 服务注册与发现工具包

> 使用 Go-kit 的 sd 包简化微服务服务注册与发现的实现。sd 包中提供如下注册和注销接口，代码如下所示：

```go
type Registrar interface { 
  Register() // 服务注册 
  Deregister() // 服务注销 
}
```

#### Service Mesh 中 Istio 服务注册与发现

> Istio 作为 Service Mesh 的落地产品之一，依托 Kubernetes 快速发展，已经成为最受欢迎的 Service Mesh 之一。Istio 在逻辑上分为数据平面和控制平面。

- 数据平面，由一组高性能的智能代理（基于 Envoy 改进的 istio-proxy）组成，它们控制和协调了被代理服务的所有网络通信，同时也负责收集和上报相关的监控数据。
- 控制平面，负责制定应用策略来控制网络流量的路由。

> Istio 由多个组件组成，核心组件及其作用为如下：

- Ingressgateway，控制外部流量访问 Istio 内部的服务。
- Egressgateway，控制 Istio 内部访问外部服务的流量。
- Pilot，负责管理服务网格内部的服务和流量策略。它将服务信息和流量控制的高级路由规则在运行时传播给 Proxy，并将特定平台的服务发现机制抽象为 Proxy 可使用的标准格式。
- Citadel，提供身份认证和凭证管理。
- Galley，负责验证、提取、处理和分发配置。
- Proxy，作为服务代理，调节所有 Service Mesh 单元的入口和出口流量。

![Istio 架构图](../../Public/images/CgqCHl82RMaAO4wvAARr5zliZpw337.png)

> 这其中 Proxy 属于数据平面，以 Sidecar 的方式与应用程序一同部署到 Pod 中，而 Pilot、Citadel 和 Galley 属于控制平面。除此之外，Istio 中还提供一些额外的插件，如 grafana、istio-tracing、kiali 和 prometheus，用于进行可视化的数据查看、流量监控和链路追踪等。

> 当然我们也可以根据实践的需求选择合适的 profile 进行安装启动，比如下面的安装命令我们使用的是 demo profile：

```bash
istioctl manifest apply --set profile=demo
```

> 上述命令以 demo profile 部署 Istio，该配置下的 Istio 能够通过可视化界面监控 Istio 中应用的方方面面。Istio 以 Sidecar 的方式在应用程序运行的 Pod 中注入 Proxy，全面接管应用程序的网络流入流出。我们可以通过标记 Kubernetes 命名空间的方式，让 Sidecar 注入器自动将 Proxy 注入在该命名空间下启动的 Pod 中，开启标记的命令如下：

```bash
kubectl label namespace default istio-injection=enabled
```

> 上述命令中，我们将 default 命名空间标记为 istio-injection。如果不想开启命令空间的标记，也可以通过 istioctl kube-inject 为 Pod 注入 Proxy Sidecar 容器。接下来，我们就为 register 服务所在的 Pod 注入 Proxy，启动命令如下：

```bash
istioctl kube-inject -f register-service.yaml | kubectl apply -f -
```

> 如果在部署 Istio 时启动了 kiali 插件，即可在 kiali 平台中查看到 register 服务的相关信息，通过以下命令即可打开 kiali 控制面板，默认账户和密码都为 admin：

```bash
istioctl dashboard kiali
```

> kiali 控制台中存在多个维度查看 Istio 中部署的应用：

- Overview，网格概述，展示 Istio 内具有服务的所有命名空间；
- Graph，服务拓扑图；
- Applications，应用维度，识别设置了 app 标签的应用；
- Workloads，负载维度，检测 Kubernetes 中的资源，包括 Deployment、Job、DaemonSet 等，无论这些资源有没有加入 Istio 中都能检测到；
- Services，服务维度，检测 Kubernetes 的 Service；
- Istio Config，配置维度，查看 Istio 相关配置类信息。

> Istio 依托 Kubernetes 的快速发展和推广，对 Kubernetes 有着极强的依赖性，其服务注册与发现的实现也主要依赖于 Kubernetes 的 Service 管理。我们可以通过以下这张图理解 Istio 的服务注册与发现。

![Istio 服务注册与发现逻辑图](../../Public/images/CgqCHl82RMaAO4wvAARr5zliZpw337.png)

> 通过该逻辑图，我们可以看到 Istio 服务注册与发现主要有以下模块参与。

- ConfigController：负责管理配置数据，包括用户配置的流量管理和路由规则。
- ServiceController：负责加载各类 ServiceRegistry，从 ServiceRegistry 中同步需要在网格中管理的服务。主要包含：①KubeServiceRegistry，从 Kubernetes 同步 Service 和 Endpoint 到 Istio；②ConsulServiceRegistry，从 Consul 中同步服务信息到 Istio；③ExternalServiceRegistry，监听 ConfigController 中的配置变化，获取 ServiceEntry 和 WorkloadEntry 资源并封装成服务数据提供给 ServiceController。
- DiscoveryServer：负责将 ConfigController 中的路由配置信息和 ServiceController 中的服务信息封装成 Proxy 可以理解的标准格式，并下发到 Proxy 中。

### 15 | 微服务间如何进行远程方法调用？

> RPC（Remote Procedure Call，远程过程调用协议），是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。RPC 只是一套协议，基于这套协议规范来实现的框架都可以称为 RPC 框架，比较典型的有 Dubbo、Thrift 和 gRPC。

#### RPC 机制和实现过程

> RPC 的机制做了经典的诠释：

> RPC 远程过程调用是指计算机 A 上的进程，调用另外一台计算机 B 上的进程的方法。其中，A 上的调用进程被挂起，而 B 上的被调用进程开始执行对应方法，并将结果返回给 A；计算机 A 接收到返回值后，调用进程继续执行。

> RPC 让程序之间的远程过程调用具有与本地调用类似的形式。比如说，程序需要读取某个文件的数据，开发人员会在代码中执行 read 系统调用来获取数据。

> 当 read 实际是本地调用时，read 函数由链接器从依赖库中提取出来，接着链接器会将它链接到该程序中。虽然 read 中执行了特殊的系统调用，但它本身依然是通过将参数压入堆栈的常规方式调用的，调用方并不知道 read 函数的具体实现和行为。

> 当 read 实际是一个远程过程时（比如调用远程文件服务器提供的方法），调用方程序中需要引入 read 的接口定义，称为客户端存根（client-stub）。远程过程 read 的客户端存根与本地方法的 read 函数类似，都执行了本地函数调用。不同的是它底层实现上不是进行操作系统调用读取本地文件来提供数据，而是将参数打包成网络消息，并将此网络消息发送到远程服务器，交由远程服务执行对应的方法，在发送完调用请求后，客户端存根随即阻塞，直到收到服务器发回的响应消息为止。

> 下图展示了远程方法调用过程中的客户端和服务端各个阶段的操作。

![RPC示意图](../../Public/images/Ciqc1F887amAAUAAAABlfyr8G5Q880.png)

> 当客户端发送请求的网络消息到达服务器时，服务器上的网络服务将其传递给服务器存根（server-stub）。服务器存根与客户端存根一一对应，是远程方法在服务端的体现，用来将网络请求传递来的数据转换为本地过程调用。服务器存根一般处于阻塞状态，等待消息输入。

> 当服务器存根收到网络消息后，服务器将方法参数从网络消息中提取出来，然后以常规方式调用服务器上对应的实现过程。从实现过程角度看，就好像是由客户端直接调用一样，参数和返回地址都位于调用堆栈中，一切都很正常。实现过程执行完相应的操作，随后用得到的结果设置到堆栈中的返回值，并根据返回地址执行方法结束操作。以 read 为例，实现过程读取本地文件数据后，将其填充到 read 函数返回值所指向的缓冲区。

> read 过程调用完后，实现过程将控制权转移给服务器存根，它将结果（缓冲区的数据）打包为网络消息，最后通过网络响应将结果返回给客户端。网络响应发送结束后，服务器存根会再次进入阻塞状态，等待下一个输入的请求。

> 客户端接收到网络消息后，客户操作系统会将该消息转发给对应的客户端存根，随后解除对客户进程的阻塞。客户端存根从阻塞状态恢复过来，将接收到的网络消息转换为调用结果，并将结果复制到客户端调用堆栈的返回结果中。当调用者在远程方法调用 read 执行完毕后重新获得控制权时，它唯一知道的是 read 返回值已经包含了所需的数据，但并不知道该 read 操作到底是在本地操作系统读取的文件数据，还是通过远程过程调用远端服务读取文件数据。

#### RPC 框架的组成

> 一个完整的 RPC 框架包含了服务注册发现、负载、容错、序列化、协议编码和网络传输等组件。不同的 RPC 框架包含的组件可能会有所不同，但是一定都包含 RPC 协议相关的组件，RPC 协议包括序列化、协议编解码器和网络传输栈，如下图所示：

![RPC框架组成图](../../Public/images/Ciqc1F887bWAQUMOAACCOORZi64063.png)

> RPC 协议一般分为公有协议和私有协议。例如，HTTP、SMPP、WebService 等都是公有协议；如果是某个公司或者组织内部自定义、自己使用的，没有被国际标准化组织接纳和认可的协议，往往划为私有协议，例如 Thrift 协议和蚂蚁金服的 Bolt 协议。

> 在协议设计上，你还需要考虑以下三个关键问题：

1. 协议包括的必要字段与主要业务负载字段。协议里设计的每个字段都应该被使用到，避免无效字段。
2. 通信功能特性的支持。比如，CRC 校验、安全校验、数据压缩机制等。
3. 协议的升级机制。毕竟是私有协议，没有长期的验证，字段新增或者修改，是有可能发生的，因此升级机制是必须考虑的。

#### RPC 和 HTTP 概念解析

![RPC 和 HTTP 关系示意图](../../Public/images/CgqCHl887caASe2qAABl7rQTtnA599.png)

> 如上图所示，HTTP 既可以和 RPC 一样作为服务间通信的解决方案，也可以作为 RPC 中通信层的传输协议（此时与之对比的是 TCP 协议）。

> HTTP 和自定义 TCP 协议都可以作为 RPC 的传输协议，二者的对比和选择也是 RPC 选型的重要考量或优化点。那么，为什么传输层协议会使用自定义的 TCP 协议呢？

> 你可能首先想到 HTTP 是无状态、无连接的，所以每次进行通信都要建立和断开连接，这会影响通信效率。

> 但实际上 HTTP 协议是支持连接池复用的，能建立一定数量的连接并且保持连接不会断开，不用频繁建立和断开连接，因此连接问题并不是优先选择自定义 TCP 协议的真正原因。

> 那真正的原因到底是什么呢？其实真正的原因就在于自定义 TCP 协议可以灵活地对协议字段进行定制，减少非必要字段的传输，减少网络开销；而 HTTP 协议则包含了过多无用的信息，比如头部等信息。

> HTTP 协议和 TCP 自定义协议的详细对比，如下图所示，可以看出自定义 TCP 协议传输相同信息时所要传递的数据量更少，所以网络通信更快，所需开销也更少。

![HTTP 协议和自定义 TCP 协议对比图](../../Public/images/Ciqc1F887d6AYkrdAACqqdChrI0328.png)

#### 常见的 PRC 框架

- Go RPC：Go 语言原生支持的 RPC 远程调用机制，简单便捷，非常适合你了解和学习 RPC 的入门框架。
- gRPC：Google 发布的开源 RPC 框架，是基于 HTTP 2.0 协议的，并支持众多常见的编程语言，它提供了强大的流式调用能力，目前已经成为最主流的 RPC 框架之一。
- Thrift：Facebook 的开源 RPC 框架，主要是一个跨语言的服务开发框架，作为老牌开源 RPC 协议，以其高性能和稳定性成为众多开源项目提供数据的方案选项。

### 16 | Go RPC 如何实现服务间通信？

#### Go RPC 原理解析

##### 1. Go RPC 服务端原理

> 服务端的 RPC 代码主要分为两个部分：①服务方法注册，包括调用注册接口，通过反射处理将方法取出，并存到 map 中；②处理网络调用，主要是监听端口、读取数据包、解码请求和调用反射处理后的方法，将返回值编码，返回给客户端。

> Register 方法中通过反射获取接口类型和值，并通过 suitableMethods 函数判断注册的 RPC 是否符合规范，最后调用 serviceMap 的 LoadOrStore(sname, s) 方法将对应 RPC 存根存放于 map 中，供之后查找。

> 接下来，我们来看一下服务端处理 RPC 请求的实现。如下图就展示了服务端 RPC 程序处理请求的过程，它会一直循环处理接收到的客户端 RPC 请求，将其交由 ReadRequestHandler 处理，然后从之前 Register 方法保存的 map 中获取到要调用的对应方法；接着从请求中解码出对应的参数，使用反射调用其方法，获取到结果后将结果编码成响应消息返回给客户端。

![RPC 服务端处理流程示意图](../../Public/images/CgqCHl8_igqAG76TAABYy-4KsGk165.png)

###### （1）接收请求

> Server 的 Accept 函数会无限循环地调用 net.Listener 的 Accept 函数来获取客户端建立连接的请求，获取到连接请求后，再使用协程来处理请求

###### （2）读取并解析请求

> ServeConn 函数会从建立的连接中读取数据，然后创建一个 gobServerCodec，并将其交由 Server 的 ServeCodec 函数处理

> ServeCodec 函数会循环地调用 readRequest 函数，读取网络连接上的字节流，解析出请求，然后开启协程执行 Server 的 call 函数，处理对应的 RPC 调用。

###### （3）执行远程方法并返回响应

> Server 的 call 函数就是通过 Func.Call 反射调用对应 RPC 过程的方法，它还会调用 send Response 将返回值发送给 RPC 客户端

##### 2. 客户端发送 RPC 请求原理

> 无论是同步调用还是异步调用，每次 RPC 请求都会生成一个 Call 对象，并使用 seq 作为 key 保存在 map 中，服务端返回响应值时再根据响应值中的 seq 从 map 中取出 Call，进行相应处理。客户端发起 RPC 调用的过程大致如下图所示。

![客户端发送和接收请求流程示意图](../../Public/images/CgqCHl8_iiGAdkeMAABxSJbVfuw198.png)

###### （1）同步调用和异步调用

> Call 方法直接调用了 Go 方法，而 Go 方法则是先创建并初始化了 Call 对象，记录下此次调用的方法、参数和返回值，并生成 DoneChannel；然后调用 Client 的 send 方法进行真正的请求发送处理，

###### （2）请求参数编码

> Client 的 send 函数首先会判断客户端实例的状态，如果处于关闭状态，则直接返回结果；否则会生成唯一的 seq 值，将 Call 保存到客户端的哈希表 pending 中，然后调用客户端编码器的WriteRequest 来编码请求并发送

###### （3）接收返回值

> 客户端的 input 函数接收服务端返回的响应值，它进行无限 for 循环，不断调用 codec 也就是 gobClientCodecd 的 ReadResponseHeader 函数，然后根据其返回数据中的 seq 来判断是否是本客户端发出请求的响应值。如果是，则获取对应的 Call 对象，并将其从 pending 哈希表中删除，继续调用 codec 的 ReadReponseBody 方法获取返回值 Reply 对象，并调用 Call 对象的 done 方法

> gobClientCodecd 的 ReadResponseHeader、ReadReponseBody 方法和上文中的 WriteRequest 类似，这里不做赘述。Call 对象的 done 方法则通过 Call 的 DoneChannel，将获得返回值的结果通知到调用层

> 客户端接收到 RPC 请求的响应后会进行其他业务逻辑操作，RPC 框架则会对进行 RPC 请求所需要的资源进行回收，下次进行 RPC 请求时则需要再次建立相应的结构体并获取对应的资源。

#### 小结

> Go 语言原生 RPC 算是个基础版本的 RPC 框架，代码精简，可扩展性高，但是只实现了 RPC 最基本的网络通信，而超时熔断、链接管理（保活与重连）、服务注册发现等功能还是欠缺的。因此还是达不到生产环境“开箱即用”的水准，不过 GitHub 就有一个基于 RPC 的功能增强版本——rpcx，支持了大部分主流 RPC 的特性。

### 17 | gRPC 和 Apache Thrift 之间如何进行选型？

#### gRPC 简介和使用

> gRPC 拥有很多特性，其中最引人注目的有以下几个方面：

- 内置流式 RPC 支持。这意味着你可以使用同一 RPC 框架来处理普通的 RPC 调用和分块进行的数据传输调用，这在很大程度上统一了网络相关的基础代码并简化了逻辑。
- 内置拦截器的支持。gRPC 提供了一种向多个服务端点添加通用功能的强大方法，这使得你可以轻松使用拦截器对所有接口进行共享的运行状况检查和身份验证。
- 内置流量控制和 TLS 支持。gRPC 是基于 HTTP/2 协议构建的，具有很多强大的特性，其中很多特性以前是必须在 Netty 上自行实现的。这使得客户端的实现更简单，并且可以轻松实现更多语言的绑定。
- 基于 ProtoBuf 进行数据序列化。ProtoBuf 是由 Google 开源的数据序列化协议，用于将数据进行序列化，在数据存储和通信协议等方面有较大规模的应用和成熟案例。gRPC 直接使用成熟的 ProtoBuf 来定义服务、接口和数据类型，其序列化性能、稳定性和兼容性得到保障。
- 底层基于 HTTP/2 标准设计。gRPC 正是基于 HTTP/2 才得以实现更多强大功能，如双向流、多复用请求和头部压缩等，从而可以节省带宽、降低 TCP 连接次数和提高 CPU 利用率等。同时，基于 HTTP/2 标准的 gRPC 还提高了云端服务和 Web 应用的性能，使得 gRPC 既能够在客户端应用，也能够在服务器端应用，从而实现客户端和服务器端的通信以及简化通信系统的构建。
- 优秀的社区支持。作为一个开源项目，gRPC 拥有良好的社区支持和维护，发展迅速，并且 gRPC 的文档也很丰富，这些对用户都很有帮助。
- 提供多种语言支持。gRPC 支持多种语言，如 C、C++、Go 、Python、Ruby、Java 、PHP 、C# 和 Node.js 等，并且能够基于 ProtoBuf 定义自动生成相应的客户端和服务端代码。目前已提供了 Java 语言版本的 gRPC-Java 和 Go 语言版本的 gRPC-Go。

> gRPC 过程调用时，服务端和客户端需要依赖共同的 proto 文件。proto 文件可以定义远程调用的接口、方法名、参数和返回值等。通过 proto 文件可以自动生成相应的客户端和服务端 RPC 代码。借助这些代码，客户端可以十分方便地发送 RPC 请求，并且服务端也可以很简单地建立 RPC 服务器、处理 RPC 请求并且将返回值作为响应发送给客户端。

> gRPC 使用一般分为三大步骤：①定义和编译 proto 文件；②客户端发送 RPC 请求；③服务端建立 RPC 服务。

#### Thrift 简介

1. 定义和编译 Thrift 文件
2. 客户端发送 RPC 请求
3. 服务端建立 RPC 服务

> 对于通信协议（TProtocol），Thrift 提供了基于文本和二进制传输协议，可选的协议有：二进制编码协议（TBinaryProtocol）、压缩的二进制编码协议（TCompactProtocol）、JSON 格式的编码协议（TJSONProtocol）和用于调试的可读编码协议（TDebugProtocol）。上面示例代码中我们使用的是默认的二进制协议，也就是 TBinaryProtocol。

> 对于传输方式（TTransport），Thrift 提供了丰富的传输方式，可选的传输方式有：最常见的阻塞式 I/O 的 TSocket、HTTP 协议传输的 THttpTransport、以 frame 为单位进行非阻塞传输的 TFramedTransport 和以内存进行传输的 TMemoryTransport 等。

> 对于服务端模型（TServer），Thrift 目前提供了：单线程服务器端使用标准的阻塞式 I/O 的 TServer、多线程服务器端使用标准的阻塞式 I/O 的 TThreadedServer 和多线程网络模型使用配有线程池的阻塞式 I/O 的 TThreadPoolServer 等。

### 18 | 案例：Go-kit 如何集成 gRPC？

```bash
protoc -I=pb --go_out=plugins=grpc:./pb --go_opt=paths=source_relative pb/user.proto
```

#### Go-kit 和 gRPC 结合的原理

- Transport 层，主要负责网络传输，例如处理HTTP、gRPC、Thrift等相关的逻辑。
- Endpoint 层，主要负责 request/response 格式的转换，以及公用拦截器相关的逻辑。作为 Go-kit 的核心，Endpoint 层采用类似洋葱的模型，提供了对日志、限流、熔断、链路追踪和服务监控等方面的扩展能力。

> Go-kit 和 gRPC 结合的关键在于需要将 gRPC 集成到 Go-kit 的 Transport 层。Go-kit 的 Transport 层用于接收用户网络请求并将其转为 Endpoint 可以处理的对象，然后交由 Endpoint 层执行，最后再将处理结果转为响应对象返回给客户端。为了完成这项工作，Transport 层需要具备两个工具方法：

- 解码器，把用户的请求内容转换为请求对象；
- 编码器，把处理结果转换为响应对象。

> gRPC 请求的处理过程如下图所示，服务端接收到一个客户端请求后，交由 grpc_transport.Handler 处理，它会调用 DecodeRequestFunc 进行解码，然后交给其 Endpoint 层转换为 Service 层能处理的对象，将返回值通过 EncodeResponseFunc 编码，最后返回给客户端。

![Go-kit 过程调用示意图](../../Public/images/CgqCHl9InuGAYQa8AABJDed6WN0517.png)

### 19 | 微服务网关如何作为服务端统一入口点？

> 在单体架构中，客户端在向服务端发起请求时，会通过类似 Nginx 的负载均衡组件获取到多个相同的应用程序实例中的一个。请求由该服务实例进行处理，服务端处理完之后返回响应给客户端。

> 而在微服务架构下，原来的单体应用拆分成了多个业务微服务。此时，直接对外暴露这些业务微服务，必然会存在一些问题。客户端直接向每个微服务发送请求，其问题主要如下：

- API 粒度的问题，客户端需求和每个微服务暴露的细粒度可能存在 API 不匹配的情况。
- 微服务之间的调用可能不仅仅基于 HTTP 的方式，还有可能使用 Thrift、gRPC 和 AMQP 消息传递协议，这些 API 无法暴露出去。
- 直接对外暴露接口，使得微服务难以重构，特别是服务数量达到一个量级，这类重构就非常困难了。

> 如上问题，解决的方案是使用微服务网关。**网关在一个 API 架构中的作用是保护、增强和控制外部请求对于 API 服务的访问。**

#### 什么是微服务网关

> 在微服务架构中，网关位于接入层之下和业务服务层之上。微服务网关是微服务架构中的一个基础服务，从面向对象设计的角度看，它与外观模式类似。

![微服务架构图](../../Public/images/Ciqc1F9PEWKAY5XRAAEa4_VCymc032.png)

> 微服务网关封装了系统内部架构，为每个客户端提供一个定制的 API，用来保护、增强和控制对于微服务的访问。换句话来讲，微服务网关就是一个处于应用程序或服务之前的系统，用来管理授权、访问控制和流量限制等，这样微服务就会被微服务网关保护起来，对所有的调用者透明。因此，隐藏在微服务网关后面的业务系统就可以更加专注于业务本身。

#### 微服务网关的功能特性

> 微服务网关的主要功能特性如下图所示：

![网关的功能特性示意图](../../Public/images/CgqCHl9PFI2AGA5BAADjWoNo2mw717.png)

> 结合该图，我们就来具体介绍下这四类功能。

- 请求接入。管理所有接入请求，作为所有 API 接口的请求入口。在生产环境中，为了保护内部系统的安全性，往往内网与外网都是隔离的，服务端应用都是运行在内网环境中，为了安全，一般不允许外部直接访问。网关可以通过校验规则和配置白名单，对外部请求进行初步过滤，这种方式更加动态灵活。
- 统一管理。可以提供统一的监控工具、配置管理和接口的 API 文档管理等基础设施。例如，统一配置日志切面，并记录对应的日志文件。
- 解耦。可以使得微服务系统的各方能够独立、自由、高效、灵活地调整，而不用担心给其他方面带来影响。软件系统的整个过程中包括不同的角色，有服务的开发提供方、服务的用户、运维人员、安全管理人员等，每个角色的职责和关注点都不同。微服务网关可以很好地解耦各方的相互依赖关系，让各个角色的用户更加专注自己的目标。
- 拦截插件。服务网关层除了处理请求的路由转发外，还需要负责认证鉴权、限流熔断、监控和安全防范等，这些功能的实现方式，往往随着业务的变化不断调整。这就要求网关层提供一套机制，可以很好地支持这种动态扩展。拦截策略提供了一个扩展点，方便通过扩展机制对请求进行一系列加工和处理。同时还可以提供统一的安全、路由和流控等公共服务组件。

#### 实战案例：自己动手实现一个网关

> API 网关根据客户端 HTTP 请求，动态查询注册中心的服务实例，通过反向代理实现对后台服务的调用。

> API 网关将符合规则的请求路由调用对应的后端服务。这里的规则可以有很多种，如 HTTP 请求的资源路径、方法、头部和参数等。

##### 1.实现思路

> 客户端向网关发起请求，网关解析请求资源路径中的信息，根据服务名称查询注册中心的服务实例；然后使用反向代理技术把客户端请求转发至后端真实的服务实例，请求执行完毕后，再把响应信息返回客户端。

> 我们设计实现的网关的功能主要包含如下几点：

- HTTP请求的规则遵循 /{serviceName}/#，否则不予通过。
- 使用 Go 提供的反向代理包 httputil.ReverseProxy 实现一个简单的反向代理，它能够对请求实现负载均衡，随机地把请求发送给服务实例。
- 使用 Consul 客户端 API 动态查询服务实例。

##### 2. 编写反向代理方法

##### 3. 编写入口方法

##### 4. 运行货运与网关服务

##### 5. 测试

### 20 | 如何进行网关选型？

> 业界有很多流行的 API 网关，比如开源的就有 Nginx、Netflix Zuul、Kong 等。Kong 有商业版，与此类似的商业版网关还有 GoKu API Gateway 和 Tyk 等。

#### Nginx

> Nginx 是于 2000 年初开发的，初衷是为解决服务端经典的C10K 问题，即在给定的内存大小内处理 10000 个以上的同时连接，在这个过程中，其他 Web 服务器的每个连接都需要占用内存，因此它们会耗尽物理内存，并在成千上万的用户同时访问一个站点时响应慢或者崩溃。Nginx 采用的是多进程事件模型、异步非阻塞的方式处理请求，因此可以同时处理成千上万个请求，并优雅地扩展到更多用户。

> 在工作方式上，Nginx 有单工作进程和多工作进程两种模式。默认情况下，Nginx 启动后，在后台以多工作进程模式运行，Nginx 的后台进程由一个 Master 进程和多个 Worker 进程组成。当然，我们也可以通过对 Nginx 进行配置，取消 Master 进程的运行，使得 Nginx 以单进程方式运行。

![Nginx 工作模型图](../../Public/images/Ciqc1F9R392AYh8gAAC7fTtAhD4549.png)

> Nginx 工作模型图如上图所示，Master 进程协调 Worker 进程，是通过进程间通信实现交互。下面我们具体介绍 Nginx 中的Master 进程和 Worker 进程：

- Master进程， 管理进程，用来协调 Worker 进程。Master 接收到信号，向 Worker 进程发送对应的信号。除此之外，Master 进程还会监控 Worker 进程的运行状态，当 Worker 进程发生异常退出时，就会重启新的 Worker 进程。
- Worker 进程， 从字面就可以知道，这是实际工作的进程。Worker 进程主要处理基本的网络事件，多个 Worker 进程之间相互独立对等，客户端请求到来时，这些进程同等竞争来自客户端的请求，最终由其中一个 Worker 进程来处理这个请求。

> 当 Nginx 标准模块和配置不能灵活地适应系统要求时，就可以考虑使用 Lua 扩展和定制 Nginx 服务。Nginx 中可以嵌入 Lua 脚本，通过将定制的 Lua 脚本部署到 Nginx，使得 Nginx 类似于一个 Web 容器。我们在开发的时候一般使用 OpenResty 来搭建开发环境，因为 OpenResty 集成了优秀的 Lua 库、第三方模块，可以方便地搭建能够处理超高并发、扩展性极高的 Web 服务。这样开发者就只需要安装 OpenResty，不需要了解 Nginx 核心和写复杂的 C/C++ 模块就可以使用 Lua 语言进行 Web 应用开发。

> Nginx 中有多个模块，这些模块根据其功能可以分为如下几种：

- 事件模块。有 Ngx_Events_Module、Ngx_Event_Core_Module 和 Ngx_Epoll_Module 等模块。事件模块提供单独的事件处理机制的框架，并提供了对各个具体事件的处理。至于使用何种 Nginx 事件处理模块，则由具体的 OS 和编译选项决定。
- 解析模块。接收客户端请求，并响应内容，比如返回静态页面的模块 Ngx_Http_Static_Module，将对应的静态文件作为响应返回给客户端。
- 输出过滤模块。对输出的内容进行处理，Nginx 中可以对响应进行修改。例如，常见的有增加 Header 或者移除 Header，这些头部可能是服务端内部的标识，因此需要我们使用输出过滤模块对响应头进行处理。
- Upstream 模块。一种特殊的解析模块，反向代理在此模块实现，Nginx 收到请求后根据配置文件，将请求转发到对应的实际后端服务器上，并返回响应给客户端。

> Upstream 模块提供的反向代理功能，使得 Nginx 具备了数据转发功能，为 Nginx 增强横向处理能力，摆脱了服务端单机处理的限制，从而使 Nginx 具备了软件级别的整合、拆分功能。使用 Nginx 的反向代理和均衡可实现负载均衡及高可用，除此之外还需要我们解决自注册和网关本身的扩展性问题。

#### Netflix Zuul

> Zuul 的核心是一系列的过滤器，这些过滤器可以完成以下功能：

![Zuul 的功能特性图](../../Public/images/Ciqc1F9R4AyAXT7kAABzyXDCdrw467.png)

> 上图中的大部分功能，你应该已经都很熟悉了，这里我们就着重讲解下静态响应处理和多区域弹性这两个功能。

- 静态响应处理：对于高并发场景下的流量压力，会存在拒绝策略，因此可以直接在服务端返回部分响应，避免转发，减少内部集群的压力。
- 多区域弹性：这个是对 AWS 的支持，服务可能部署在多个 AWS Region，多区域弹性使得转发的实际服务能够靠近用户所在的地区。

> 上面描述的一些特性是 Nginx 不具备的，这是因为Zuul设计之初的目标就不仅仅是作为类似于 Nginx 的反向代理，而是意在解决云环境下的种种问题。Zuul 目前有两个大的版本：Zuul1 和 Zuul2。

- Zuul1 的设计比较简单，本质上就是一个同步 Servlet，如图所示。Spring Cloud 中集成的 Zuul 版本， 采用的是 Tomcat 容器，使用的是传统的 Servlet IO 处理模型。Zuul 的多线程阻塞模型，即一个线程处理一次连接请求，这种方式在内部延迟严重、设备故障较多情况下会引起存活的连接增多和线程增加的情况发生。

![Zuul1 的工作原理图](../../Public/images/Ciqc1F9R4COANCgcAACDJ9W7E_8013.png)

> Netflix 发布的 Zuul2 有重大的更新，采用异步非阻塞的方式处理请求，每个 CPU 核对应一个线程，用来处理所有的请求和响应。请求和响应的生命周期是通过事件和回调来处理的，这种方式减少了线程数量，因此开销较小。

![Zuul2 的工作原理图](../../Public/images/Ciqc1F9R4DuAZIntAAB3AwAig7w452.png)

> 非阻塞模式可以接受的连接数大大增加，这种模式下，请求到达则进入处理队列，这个队列的容量可以设得很大。在请求超时之前，队列中的请求都会被依次处理。又由于数据被存储在同一个 CPU 里，Zuul2可以复用 CPU 级别的缓存，这种方式较Zuul1线程切换来说轻量级很多，消耗会较小。

#### Kong

> Kong 由 Mashape 公司开源，是一款高性能高可用的微服务网关，是基于 OpenResty（Nginx + Lua模块）编写的高可用服务网关，由于 Kong 是基于 Nginx 的，因此可以很轻松地水平扩展 Kong 服务器。通过负载均衡把请求转发给各个服务端实例，来应对大批量的网络请求。

![Kong 的基本运行图](../../Public/images/CgqCHl9R4EOAWxLXAAC95rLtc-U861.png)

> Kong 的基本运行情况如上图所示，客户端请求到达 Kong 网关之后，经过一系列插件的处理之后，才会将请求转发给指定的后端服务。Kong 主要包括 Kong Server、Apache Cassandra/PostgreSQL 和 Kong Dashboard三个组件。

- Kong Server：基于Nginx的服务器，用来接收 API 请求。
- Apache Cassandra/PostgreSQL：用来存储操作数据。
- KongDashboard：官方提供的 Kong 图形化管理工具，如果不使用图形化界面的话，也可以直接使用 Restful 方式管理 Admin API。

> 这三部分组成了完整的 Kong 网关。Kong 本身也是基于 Nginx 的，所以在性能和稳定性上都没有问题。Kong 网关具有以下的特性：

- 可扩展性。通过简单添加更多的服务器，可以轻松进行横向扩展，这意味着平台可以在一个较低负载的情况下处理任何请求。
- 模块化。这是 Kong 的一大特性，可以通过添加新的插件进行扩展，且插件可定制开发，可以通过 Kong Dashboard 或者 RESTful Admin API 进行配置。
- 可在任何基础架构上运行。可以在云环境或内部网络环境中部署 Kong，对云原生环境有着天然地支持。

> 基于 Kong 插件机制，使用 Lua 脚本开发，可以便捷地进行功能定制，这些插件（可能是 0 个或者 N 个）在 API 请求响应循环的生命周期中被执行。Kong 中提供的基础插件包括：HTTP 基本认证、Key Authentication、CORS（Cross-Origin Resource Sharing，跨域资源共享）、OAuth2.0 认证、文件日志、网关限流、请求的路由转发和服务监控等。我们会在下面的课时重点介绍这其中的几款插件。

#### 网关对比

![网关对比](../../Public/images/Ciqc1F9R4GCAPyX4AAFMzQx0Fy8655.png)

### 21 | 案例：如何使用 Kong 进行网关业务化定制？

#### 为什么使用 Kong

> 在用 Kong 进行实践之前，我们得先介绍一些 Kong 中常用的术语，因为这些术语在实践中会经常用得到。

- Route：请求的转发规则，按照 Hostname 和 PATH，将请求转发给 Service。
- Services：多个 Upstream 的集合，是 Route 的转发目标。
- Consumer：API 的用户，记录用户信息。
- Plugin：插件，可以是全局的，也可以绑定到 Service、Router 或者 Consumer。
- Certificate：HTTPS 配置的证书。
- SNI：域名与 Certificate 的绑定，指定了一个域名对应的 HTTPS 证书。
- Upstream：上游对象用来表示虚拟主机名，拥有多个服务（目标）时，会对请求进行负载均衡。
- Target：最终处理请求的 Backend 服务。

#### 安装实践

##### 1. 创建服务

> 服务 Services 是上游服务的抽象，可以是一个应用，或者具体某个接口。Kong 提供了管理接口，我们可以通过请求 8001 管理接口直接创建，也可以通过安装的管理界面，这二者的实现效果是一样的。

```bash
curl -i -X POST \
--url http://localhost:8001/services/ \
--data 'name=aoho-blog' \
--data 'url=http://blueskykong.com/'
```

##### 2. 创建路由

> 创建好服务之后，我们需要创建具体的 API 路由。路由是请求的转发规则，根据 Hostname 和 PATH，将请求转发。

```bash
curl -i -X POST \
--url http://localhost:8001/services/aoho-blog/routes \
--data 'hosts[]=blueskykong.com' \
--data 'paths[]=/api/blog'
```

#### 安装 Kong 插件

##### 1. JWT 认证插件

> JWT（JSON Web Token）是一种流行的跨域身份验证解决方案。作为一个开放的标准（RFC 7519），JWT 定义了一种简洁的、自包含的方法用于通信双方之间以 JSON 对象的形式安全地传递信息。因为数字签名的存在，这些信息是可信的。

> JWT 最大的优点就是能让业务无状态化，让 Token 作为业务请求的必须信息随着请求一并传输过来，服务端不用再去存储 session 信息，尤其是在分布式系统中。Kong 提供了 JWT 认证插件，用以验证包含 HS256 或 RS256 签名的 JWT 请求。每个消费者都将拥有 JWT 凭证（公钥和密钥），这些凭证必须用于签署其 JWT。JWT 令牌可以通过请求字符串、Cookie 或者认证头部传递，Kong 将会验证令牌的签名，通过则转发，否则直接丢弃请求。

> 我们在前面配置的路由基础上，增加 JWT 认证插件：

```bash
curl -X POST http://localhost:8001/routes/e33d6aeb-4f35-4219-86c2- a41e879eda36/plugins \
--data "name=jwt"
```

> 在增加了 JWT 插件之后，就没法直接访问 /api/blog 接口了，接口返回 "message": "Unauthorized"，提示客户端要访问的话则需要提供 JWT 的认证信息。因此，我们需要创建用户：

```bash
curl -i -X POST \
--url http://localhost:8001/consumers/  \
--data "username=aoho"
```

> 创建好用户之后，需要获取用户 JWT 凭证，执行如下的调用：

```bash
curl -i -X POST \
--url http://localhost:8001/consumers/aoho/jwt \
--header "Content-Type: application/x-www-form-urlencoded"
```

> 使用 key 和 secret 在 https://jwt.io 可以生成 JWT 凭证信息。在实际的使用过程中，我们通过编码实现，此处为了演示使用网页工具生成 Token。

```bash
curl --location --request GET 'http://localhost:8000/api/blog' \
--header 'Host: blueskykong.com' \
--header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzUXJWWWpJTklGOVVZQnhwYWFXcjhCd3pGam15QzdSayJ9.KBv3oT3xcIaWwUlgpYJ-wrtThag0UXwgld3i2qxoTGs'
```

##### 2.Prometheus 可视化监控

> 作为新一代的监控框架，Prometheus 适用于记录时间序列数据，并且它还具有强大的多维度数据模型、灵活而强大的查询语句、易于管理和伸缩等特点。

> Kong 官方提供的 Prometheus 插件，可用的 metric（指标）有如下：

- 状态码。上游服务返回的 HTTP 状态码。
- 时延柱状图。Kong 中的时延都将被记录，包括请求（完整请求的时延）、Kong（Kong 用来路由、验证和运行其他插件所花费的时间）和上游（上游服务所花费时间来响应请求）。
- Bandwidth。流经 Kong 的总带宽（出口/入口）。
- DB 可达性。Kong 节点是否能访问其 DB。
- Connections。各种 Nginx 连接指标，如 Active、读取、写入和接收连接。

> 我们在 Service 为 aoho-blog 安装 Prometheus 插件：

```bash
curl -X POST http://localhost:8001/services/aoho-blog/plugins \
--data "name=prometheus" 
```

> 通过访问/metrics接口返回收集度量数据：

```bash
curl -i http://localhost:8001/metrics
```

> Prometheus 插件导出的度量标准，可以在 Grafana（一个跨平台的开源的度量分析和可视化工具）中绘制，“Prometheus + Grafana” 的组合是目前较为流行的监控系统。

##### 3. 链路追踪 Zipkin 插件

> Zipkin 是由 Twitter 开源的分布式实时链路追踪组件。 Zipkin 收集来自各个异构系统的实时监控数据，用来追踪与分析微服务架构下的请求，应用系统则需要向 Zipkin 报告链路信息。Kong 的 Zipkin 插件将 Kong 作为 zipkin-client，zipkin-client 组装好 Zipkin 需要的数据包发送到 zipkin-server。Zipkin 插件会将请求打上如下标签，并推送到 Zipkin 服务端：

- span.kind (sent to Zipkin as “kind”)
- http.method
- http.status_code
- http.url
- peer.ipv4
- peer.ipv6
- peer.port
- peer.hostname
- peer.service

> 首先开启 Zipkin 插件，将插件绑定到路由上（这里可以绑定为全局的插件）。

```bash
curl -X POST http://localhost:8001/routes/e33d6aeb-4f35-4219-86c2-a41e879eda36/plugins \
    --data "name=zipkin"  \
    --data "config.http_endpoint=http://localhost:9411/api/v2/spans" \
    --data "config.sample_ratio=1"
```

> 如上配置了 Zipkin Collector 的地址和采样率，为了测试效果明显，设置采样率为 100%，但在实际生产环境中还是要谨慎使用 100% 的采样率配置，采样率对系统吞吐量会有影响。


### 22 | 如何保障分布式系统的高可用性？（上）

#### 系统可用性指标

> 系统可用性指标是衡量分布式系统高可用性的重要因素，它通常是指系统可用时间与总运行时间之比，即`Availability=MTTF/(MTTF+MTTR)`。

> 其中，MTTF（Mean Time To Failure）是指平均故障前的时间，一般是指系统正常运行的时间。系统的可靠性越高，MTTF 越长，即系统正常运行的时间越长。

> MTTR（Mean Time To Recovery）是指平均修复时间，即从故障出现到故障修复的这段时间，也就是系统不可用的时间。MTTR 越短说明系统的可用性越高。

> 系统可用性指标可以通过下表的 9999 标准衡量，现在普遍要求至少 2 个 9，最好 4 个 9 以上：

![系统可用性指标](../../Public/images/CgqCHl9bTpyAJGKxAAELzFEHw78546.png)

#### 冗余设计

> 目前常见的冗余设计有主从设计和对等治理设计，其中主从设计又可以细分为一主多从、多主多从。

> 冗余设计中一个不可避免的问题是考虑分布式系统中的数据一致性，多个节点中冗余的数据追求强一致性还是最终一致性。即使节点提供无状态服务，也需要借助外部服务，比如数据库、分布式缓存等维护数据状态。

> CAP 是描述分布式系统下节点数据同步的基本原则，分别指：

- Consistency，数据强一致性，各个节点中对于同一份数据在任意时刻都是一致的；
- Availablity，可用性，系统在任何情况下接收到客户端请求后，都能够给出响应；
- Partition Tolerance，分区容忍性，系统允许节点网络通信失败。

> 分布式系统一般基于网络进行数据通信，所有 P 是必须满足。但是满足数据强一致性的系统无法保证可用性，最典型的例子就是 ZooKeeper。

> ZooKeeper 采用主从设计，服务集群由 Leader、Follower 和 Observer 三种节点角色组成，它们的职责如下表所示：

![ZooKeeper角色与职责](../../Public/images/CgqCHl9bTq2AVQZgAADxs8PW6qM628.png)

> 在 ZooKeeper 集群中，由于只有 Leader 角色的节点具备写数据的能力，所以当 Leader 节点宕机时，在新的 Leader 节点没有被选举出来之前，集群的写能力都是不可用的。在这样的情况下，虽然 ZooKeeper 保证了集群数据的强一致性，但是此时集群无法响应客户端的写请求，即不满足 A 可用性原则。

> 对等治理设计中比较优秀的业内体现为 Netiflx 开源的Eureka 服务注册和发现组件。Eureka 集群由 Eureka Client 和 Eureka Server 两种节点角色组成，其中 Eureka Client 是指服务实例使用的服务注册和发现的客户端，各服务实例使用它来与 Eureka Server 进行通信，主要用于向 Eureka Server 请求服务注册表中的数据和注册自身服务实例信息； Eureka Server 作为服务注册中心，在注册表中存储了各服务实例注册的服务实例信息，并定时与服务实例维持心跳，剔除掉注册表中长时间心跳失败的服务实例。Eureka Server 采用多实例的方式保证高可用性部署。

> 每一个 Eureka Server 都是对等的数据节点，Eureka Client 可以向任意的 Eureka Server 发起服务注册请求和服务发现请求。Eureka Server 之间的数据通过异步 HTTP 的方式同步，由于网络的不可靠性，不同 Eureka Server 中的服务实例数据不能保证在任意时间节点都相等，只能保证在 SLA 承诺时间内达到数据的最终一致性。Eureka点对点对等的设计保证了服务注册与发现中心的高可用性，但是由于 Eureka Server 数据同步的不可靠性，数据的强一致性降级为数据的最终一致性。

#### 熔断设计

> 在分布式系统中，一次完整的请求可能需要经过多个服务模块的通力合作，请求在多个服务中传递，服务对服务的调用会产生新的请求，这些请求共同组成了这次请求的调用链。当调用链中的某个环节，特别是下游服务不可用时，将会导致上游服务调用方不可用，最终将这种不可用的影响扩大到整个系统，导致整个分布式系统的不可用，引发服务雪崩现象。

> 为了避免这种情况，在下游服务不可用时，保护上游服务的可用性显得极其重要。对此，我们可以参考电路系统的断路器机制，在必要的时候“壮士断腕”，当下游服务因为过载或者故障出现各种调用失败或者调用超时现象时，及时“熔断”服务调用方和服务提供方的调用链，保护服务调用方资源，防止服务雪崩现象的出现。

> 断路器的基本设计图如下所示，由关闭、打开、半开三种状态组成。

![断路器的基本设计图](../../Public/images/CgqCHl9bJX6AYR6WAACD8asiP4k125.png)

- 关闭（Closed）状态：此时服务调用者可以调用服务提供者。断路器中使用失败计数器周期性统计请求失败次数和请求总次数的比例，如果最近失败频率超过了周期时间内允许失败的阈值，则切换到打开（Open）状态。比如在查询历史订单数据时，订单服务出现短时间的宕机，该段时间内的查询历史订单的请求都会失败，100% 的调用失败率超过了断路器中的预设的失败阈值 50%，那么断路器就会打开。在关闭状态下，失败计数器基于时间周期运作，会在每个统计周期开始前自动重置，防止某次偶然错误导致断路器进入打开状态。
- 打开（Open）状态：在该状态下，对应用程序的请求会立即返回错误响应或者执行预设的失败降级逻辑，而不调用服务提供者。接着刚才调用订单服务的例子，在断路器处于打开状态时，所有查询历史订单的请求都会执行预设的失败降级逻辑，直接返回“系统繁忙，稍后再试”的提示语，避免服务调用者浪费资源进行无效的请求。断路器进入打开状态后会启动超时计时器，在计时器到达后，断路器进入半开状态，给此时不可用的服务提供者一定的时间进行恢复。
- 半开（Half-Open）状态：允许应用程序一定数量的请求去调用服务。如果这些请求对服务的调用成功，那么可以认为之前导致调用失败的错误已经修正，此时断路器切换到关闭状态，同时将失败计数器重置。如果这一定数量的请求存在调用失败的情况，则认为导致之前调用失败的问题仍然存在，断路器切回到打开状态，并重置超时计时器来给系统一定的时间修正错误。半开状态能够有效防止正在恢复中的服务被突然而来的大量请求再次打垮。比如订单服务在超时计时器达到之前还没修复好，从服务调用者过来的调用流量可能会破坏原先的问题环境，导致订单服务的问题排查处理更困难。半开状态也给服务调用恢复正常的机会，如果此时订单服务修复成功，半开状态尝试的请求都能够正常返回，那么就关闭断路器，查询历史订单数据的请求都恢复正常处理。

> 使用断路器设计模式，能够有效地保护服务调用方的稳定性，它能够避免服务调用者频繁调用可能失败的服务提供者，防止服务调用者浪费 CPU 周期、线程和 IO 资源等，提高服务整体的可用性。

### 23 | 如何保障分布式系统的高可用性？（下）

#### 限流设计

- 拒绝服务
- 服务降级
- 优先级请求
- 延时处理
- 弹性伸缩

> 限流设计最主要的思想是保证系统处理自身承载能力内的请求访问，拒绝或者延缓处理过量的流量，而这种思想主要依赖于它的限流算法。那接下来我们介绍两种常用的限流算法：漏桶算法和令牌桶算法。

> 漏桶算法是网络世界中流量整形或速率限制时经常使用的一种算法，它的主要目的是控制数据进入系统的速率，平滑对系统的突发流量，为系统提供一个稳定的请求流量。

> 令牌桶算法则是一个存放固定容量令牌的桶，按照固定速率往桶里添加令牌。桶中存放的令牌数有最大上限，超出之后就被丢弃。一般来说令牌桶内令牌数量上限就是系统负载能力的上限，不建议超过太多。当流量或者网络请求到达时，每个请求都要获取一个令牌，如果能够从令牌桶中获取到令牌，请求将被系统处理，被获取到的令牌也会从令牌桶中移除；如果获取不到令牌，该请求就要被限流，要么直接丢弃，要么在缓冲区等待（如下图所示）。令牌桶限制了请求流量的平均流入速率，令牌以一定的速率添加到桶内，只要桶里有足够的令牌，所有的请求就能流入系统中被处理，这能够应对一定程度的突发巨量流量。

![令牌桶算法示意图](../../Public/images/Ciqc1F9h0BaANqlVAACr8W_zBko547.png)

#### 降级设计

> 在应对大流量冲击时，可以尝试对请求的处理流程进行裁剪，去除或者异步化非关键流程的次要功能，保证主流程功能正常运转。

> 一般来说，降级时可以暂时“牺牲”的有：

- 降低一致性。从数据强一致性变成最终一致性，比如说原本数据实时同步方式可以降级为异步同步，从而系统有更多的资源处理响应更多请求。
- 关闭非关键服务。关闭不重要功能的服务，从而释放出更多的资源。
- 简化功能。把一些功能简化掉，比如，简化业务流程，或是不再返回全量数据，只返回部分数据。也可以使用缓存的方式，返回预设的缓存数据或者静态数据，不执行具体的业务数据查询处理。

#### 无状态设计

> 在分布式系统设计中，倡导使用无状态化的方式设计开发服务模块。这里“无状态”的意思是指对于功能相同的服务模块，在服务内部不维护任何的数据状态，只会根据请求中携带的业务数据从外部服务比如数据库、分布式缓存中查询相关数据进行处理，这样能够保证请求到任意服务实例中处理结果都是一致的。

> 无状态设计的服务模块可以简单通过多实例部署的方式进行横向扩展，各服务实例完全对等，可以有效提高服务集群的吞吐量和可用性。但是如此一来，服务处理的性能瓶颈就可能出现在提供数据状态一致性的外部服务中。

#### 幂等性设计

> 幂等性设计是指系统对于相同的请求，一次和多次请求获取到的结果都是一样的。幂等性设计对分布式系统中的超时重试、系统恢复有重要的意义，它能够保证重复调用不会产生错误，保证系统的可用性。一般我们认为声明为幂等性的接口或者服务出现调用失败是常态，由于幂等性的原因，调用方可以在调用失败后放心进行重新请求。

> 举个简单的例子，在一笔订单的支付中，订单服务向支付服务请求支付接口，由于网络抖动或者其他未知的因素导致请求没能及时返回，那么此时订单服务并不了解此次支付是否成功。如果支付接口是幂等性的，那我们就可以放心使用同一笔订单号重新请求支付，如果上次支付请求已经成功，将会返回支付成功；如果上次支付请求未成功，将会重新进行金额扣费。这样就能保证请求的正确进行，避免重复扣费的错误。

#### 超时设计

> 鉴于目前网络传播的不稳定性，在服务调用的过程中，很容易出现网络包丢失的现象。如果在服务调用者发起调用请求处理结果时出现网络丢包，在请求结果返回之前，服务调用者的调用线程会一直被操作系统挂起；或者服务提供者处理时间过长，迟迟没返回结果，服务调用者的调用线程也会被同样挂起。当服务调用者中出现大量的这样被挂起的服务调用时，服务调用者中的线程资源就可能被耗尽，导致服务调用者无法创建新的线程处理其他请求。这时就需要超时设计了。

> 超时设计是指给服务调用添加一个超时计时器，在超时计时器到达之后，调用结果还没返回，就由服务调用者主动结束调用，关闭连接，释放资源。通过超时设计能够有效减少系统等待时间过长的服务调用，使服务调用者有更多的资源处理其他请求，提高可用性。但是需注意的是，要根据下游服务的处理和响应能力合理设置超时时间的长短，过短将会导致服务调用者难以获取到处理结果，过长将会导致超时设计失去意义。

#### 重试设计

> 在很多时候，由于网络不可靠或者服务提供者宕机，服务调用者的调用很可能会失败。如果此时服务调用者中存在一定的重试机制，就能够在一定程度上减少服务失败的概率，提高服务可用性。

> 使用重试设计的时候需要注意以下问题：

- 待重试的服务接口是否为幂等性。对于某些超时请求，请求可能在服务提供者中执行成功了，但是返回结果却在网络传输中丢失了，此时若重复调用非幂等性服务接口就很可能会导致额外的系统错误。
- 服务提供者是否只是临时不可用。对于无法快速恢复的服务提供者或者网络无法立即恢复的情况下，盲目的重试只会使情况更加糟糕，无脑地消耗服务调用方的 CPU 、线程和网络 IO 资源，过多的重试请求甚至可能会把不稳定的服务提供者打垮。在这种情况下建议你结合熔断设计对服务调用方进行保护。

#### 接口缓存

> 接口缓存是应对大并发量请求，降低接口响应时间，提高系统吞吐量的有效手段。基本原理是在系统内部，对于某部分请求参数和请求路径完成相同的请求结果进行缓存，在周期时间内，这部分相同的请求结果将会直接从缓存中读取，减少业务处理过程的负载。

> 最简单的例子是在一些在线大数据查询系统中，查询系统会将周期时间内系统查询条件相同的查询结果进行缓存，加快访问速度。

> 但接口缓存同样有着它不适用的场景。接口缓存牺牲了数据的强一致性，因为它返回的过去某个时间节点的数据缓存，并非实时数据，这对于实时性要求高的系统并不适用。另外，接口缓存加快的是相同请求的请求速率，这对于请求差异化较大的系统同样无能为力，过多的缓存反而会大量浪费系统内存等资源。

#### 实时监控和度量

> 由于分布式中服务节点众多，问题的定位变得异常复杂，对此建议对每台服务器资源使用情况和服务实例的性能指标进行实时监控和度量。最常见的方式是健康检查，通过定时调用服务提供给健康检查接口判断服务是否可用。

> 目前业内也有开源的监控系统 Prometheus，它监控各个服务实例的运行指标，并根据预设的阈值自动报警，及时通知相关开发运维人员进行处理。

#### 常规化维护

> 定期清理系统的无用代码，及时进行代码评审，处理代码中 bad smell，对于无状态服务可以定期重启服务器减少内存碎片和防止内存泄漏……这些都是非常有效的提高系统可用性的运维手段。

### 24 | 如何实现熔断机制？

#### 分布式系统中的服务雪崩

> 在分布式系统中，由于业务上的划分，一次完整的请求可能需要借助好几个不同的模块协力完成，在微服务架构中就是需要多个服务实例协力完成。请求会在这些服务实例中传递，服务之间的调用会产生新的请求，它们共同组成一次服务调用链，关系如下图所示：

![微服务服务调用链示意图](../../Public/images/CgqCHl9hoYyAd7J7AAAu6XVpTzE568.png)

> 服务雪崩是指当调用链的某个环节（特别是服务提供方服务）不可用时，将会导致其上游环节不可用，并最终将这种影响扩大到整个系统中，导致整个系统的不可用。如下图所示：

![服务雪崩示意图](../../Public/images/CgqCHl9hoZqAaUmRAABYO2_cRvk414.png)

> 为了避免服务雪崩现象的出现，我们就需要及时“壮士断腕”，在必要的时候暂时切断对异常服务提供者的调用，以保证部分服务可用和整体系统的稳定性。这时断路器（Circuit Breaker）就该登场了。

![断路器](../../Public/images/Ciqc1F9hoaiANJYgAAA_xMaFiWc181.png)

#### Hystrix 简介

> Hystrix 是 Netflix 开源的一个优秀的服务间断路器。它能够在服务提供者出现故障时，隔离服务调用者和服务提供者，防止服务级联失败；同时提供失败回滚逻辑，使系统快速从异常中恢复。

> Hystrix 完美地实现了断路器模式，同时还提供了信号量和线程隔离的方式保护服务调用者的线程资源，对延迟和失败提供强大的容错能力，进而保护和控制系统。

#### Hystrix 原理

> 看完 Hystrix 使用的简单案例后，我们再来了解一下 Hystrix 断路器的相关原理，分析一下 Hystrix.Do 的关键实现。

![Hystrix 断路器原理示意图](../../Public/images/Ciqc1F9hodmASLFTAADzDRuBp1g798.png)

> 从上述的流程图中，可以看到：

- Hystrix.Do 或 Go 函数执行时，都会调用 Hystrix.Doc 函数，然后交由对应的 HystrixCommand 处理。
- HystrixCommand 会首先调用 allowRequest 函数判断当前是否处在熔断状态中，如果不是则直接放行，执行包装定义的代码逻辑；如果是的话，还要看是否到达预定熔断时长，如果熔断时长到了，也是放行，否则直接执行预先设定的错误逻辑。
- HystrixCommand 执行结束前都会调用 markSuccess(duration) 或 markFailure(duration) 函数，统计一下在一定的 duration（时间间隔）内有多少调用是成功的，另有多少调用是失败的。
- isOpen 函数会判断当前是否需要打开断路器，计算一下本个周期内的请求整体错误率，如果高于一个阈值，那么打开熔断，否则关闭。
- Hystrix 会在内存中为每个 HystrixCommand 维护一个数组，其记录着每一个周期的请求结果的统计。该数组只能维护固定数量的周期数据，超过一定时间的周期数据会被删除掉，这样才能添加新的周期数据进入数组。

### 25 | 如何实现接口限流和降级？

#### 限流和降级

##### 1. 应用的服务定位不同

![系统服务定位和分类示意图](../../Public/images/CgqCHl9hpCCAZePCAABZjumIw_A149.png)

> 如上图所示，外部请求通过网关直接访问上流服务 A，服务 A、服务 B 和服务 C 是核心链路服务，服务 D 是次要链路服务，服务 E 是外部服务。

> 其中，外部请求的数量往往无法控制，比如双 11 抢购和微博热点事件，可能会有巨量瞬时流量，所以直面不可控的外部请求的服务 A 往往需要进行限流，需要根据整条核心链路经过压测计算出来的处理上限，过滤掉部分请求。

> 除了服务 A 需要限流外，一般来说，服务 B、服务 C 和服务 D 都不需要限流。而服务 E 是外部服务，在另外一个系统中也相当于服务 A 的地位，直面外部请求，所以理论上它也是需要进行限流操作的，以防止上游系统的请求将其冲垮。

> 对于服务 B 来说，它除了负责自身业务外，还需要协调调用服务 C、服务 D 和服务 E，当流量过大系统逐渐无法应对时，往往就会引入降级，关闭掉对次要链路中服务 D 的调用，腾出更多的服务器资源给核心链路的服务。

> 服务 B 需要调用外部服务 E，它是一个稳定性未知的外部服务，相当于一个黑盒，所以一般服务 B 需要断路器来提供熔断机制以保护自己不被外部服务 E 拖垮。

##### 2. 效果不同

> 限流方案能预防突发的大流量，保护系统不被冲垮，但是其在处理瞬时流量时，大多数时候会拒绝掉系统无法处理的过量流量，服务的处理能力并没有过多改变，这就可能会导致拒绝掉一些关键业务请求的尴尬情况发生。

> 而降级设计能够暂时提高系统某些关键服务的处理能力，从而承载更多的请求访问，当然它会牺牲其他次要功能的资源。

#### 限流案例

> 关于限流，我们可以直接使用 Golang 标准库自带的 x/time/rate 组件。

> rate 组件是基于经典的令牌桶（Token Bucket）算法实现的。

> 而限流器就相当于一个这样的令牌桶，每次处理请求时都需要从限流器这里获得一个令牌，如果没有则一直等待。所以，要使用 rate 组件实现限流功能，我们先要构造一个限流器。rate 提供了 NewLimiter 方法来构造限流器，它有两个参数：

- 第一个参数代表系统每秒钟向令牌桶中放入多少个令牌，也就是限流器平稳状态下每秒可以允许多少请求通过，它的参数类型是 Limit，是 float64 类型的别名；
- 第二个参数代表令牌桶的上限或者整体大小，也就是限流器允许多大的瞬时请求流量通过。

```go
// 使用x/time/rate创建限流中间件
limiter := rate.NewLimiter(10, 20)
```

> 除了直接指定每秒产生的令牌个数外，还可以用 Every 方法来指定向令牌桶中放置令牌的间隔，例如：

```go
limit := Every(100 * time.Millisecond);
limiter := NewLimiter(limit, 1);
```

> 以上就表示每 100ms 往桶中放一个令牌，也就是说 1 秒钟可以产生 10 个令牌。

> Limiter 提供了三类方法供用户消费令牌，用户可以每次消费一个令牌，也可以一次性消费多个令牌，这里我们就只展示其中的 Allow 类方法的使用（如下示例代码），剩下的 Wait 和 Reserve 类方法跟 Allow 类方法的使用比较类似，

```go
if !limit.Allow() {
    // Allow返回false，表示桶内不足一个令牌，应该被限流，默认返回 ErrLimiExceed 异常
    return nil, errors.New("ErrLimitExceed")
}
// Allow 返回true，则正常执行业务逻辑
return doRealJob(params)
```

#### 降级案例

> 降级的核心目的就是要保证服务基本可用，一般可以通过配置中心配置或人工配置一些关键数据去进行降级。同时，需要注意的是，降级也需要根据系统的吞吐量、响应时间、可用率等条件进行自动或手动降级。

> 一般来说，降级的手段有很多，比如有一种降级方案就是通过断路器打开和被限流时返回的默认固化的数据或者处理逻辑来实现的。除此之外，还可以使用配置中心提供动态开关的手段进行服务降级。

![降级示意图](../../Public/images/CgqCHl9hpE2AMoEtAABJPMCu7CE561.png)

> 动态开关方案需要依赖配置中心，并且配置中心和服务需要提供如下一些功能。

- 启动主动拉取配置：服务启动时拉取配置相关数据，并缓存在本地。
- 发布订阅配置：当决定降级时，配置中心中保存的配置数据会被修改，服务需要感知该变更并主动更新本地缓存。
- 定时拉取配置：可以解决订阅配置失效等极端场景问题。

> 分布式系统将 etcd 用作配置管理、服务发现和协调分布式工作的一致键值存储组件。许多组织在生产系统上使用 etcd，例如容器调度程序、服务发现和分布式数据存储。所以，可以将服务降级相关的开关配置存储到 etcd 中，并供业务服务拉取和订阅。

### 26 | 案例：如何通过 Service Mesh 实现熔断和限流？

#### Service Mesh 基本原理和优缺点

![Hystrix 和 Istio 的使用区别](../../Public/images/CgqCHl9q61aAEwYWAAB6vty4Sd8250.png)

> 下面，我们就以熔断为例，对比一下使用Service Mesh 方式和使用 Hystrix 类通用组件的优缺点。

- Service Mesh 是“黑盒”的熔断工具，而 Hystrix 等通用组件则可以被看成是“白盒”的熔断工具。
具体判断标准是看二者到底是通过外部监控系统监控服务网络请求情况，还是在服务内部监控服务的请求响应情况。如下图所示，Service Mesh 是独立于服务实例之外的存在，通过代理所有服务实例的网络请求来监控响应信息实现熔断机制；而 Hystrix 等通用组件，则是在服务实例中发挥作用，作为服务实例进行网络请求的基础层之一，来监控网络请求信息。

- Service Mesh 是无侵入地实现熔断，而 Hystrix 等通用组件则需要侵入具体服务业务代码中。
Service Mesh 通过在网络层面配置熔断策略，可以在不更改服务代码的情况下为服务实例提供熔断保护，其对服务来说几乎是无感和透明的。通过这个方法，Service Mesh 适用于任何语言或者框架开发的服务，并且开发人员也不必为每个服务单独配置或重新编程，减少了开发人员的工作量。Hystrix等通用组件需要更改服务的具体代码来引入其第三方依赖库，这些通用组件目前也只支持几类较为主流的编程语言，适用性不如 Service Mesh高。除此之外，开启或关闭熔断机制需要重新部署或者重启服务实例，这对高可用的服务系统来说代价很高。

- Service Mesh 熔断机制能提供的回退行为较为有限，而 Hystrix 等通用组件提供了较为丰富的回退实现。
正是因为 Service Mesh 是无侵入的，所以它只能在系统触发熔断机制时，通过 Proxy 返回 HTTP 503 错误，期望应用服务在应对503错误时进行各类回退操作。 而 Hystrix等通用组件在其 API 层面提供了多种的回退操作实现，这样可以实现各种不同类型的回退操作，比如单一的默认值、使用缓存结果或者去调用其他服务等。

#### Istio 的熔断限流实践

> Istio使用目标规则中的 TrafficPolicy 属性来配置熔断和限流，其中 connectionPool 属性配置限流方式，outlierDetection 配置异常检测的熔断方式。下面，我们来分别看一下这二者是如何配置的。

> ConnectionPool 下有 TCP 和 HTTP 两个类别的配置，二者相互协作，为服务提供有关限流的配置。

> TCP 相关的基础配置有maxConnections 和 connectTimeout。

- maxConnections 表示到目标服务最大的 HTTP1/TCP 连接数量。它只会限制基于HTTP1.1协议的连接，不会影响基于HTTP2的连接，因为 HTTP2 协议只建立一次连接。
- connectTimeout 表示建立 TCP 连接时的超时时间，默认单位是秒。超出该时间，则连接会被自动断开。

> HTTP下的配置包括http1MaxPendingRequests、http2MaxRequests 和 maxRequestsPerConnection 三种。

- http1MaxPendingRequests 表示HTTP 请求处于pending状态下的最大请求数，也就是目标服务最多可以同时处理多少个 HTTP 请求，默认是1024 个。
- http2MaxRequests 表示目标服务最大的 HTTP2 请求数量，默认是1024。
- maxRequestsPerConnection 表示每个 TCP 连接可以被多少个请求复用，如果将这一参数设置为 1，则会禁止 keepalive 特性。

> OutlierDetection 下相关的配置项涉及服务的熔断机制，具体有如下几个基础配置。

- consecutiveErrors 表示如果目标服务连续返回多少次错误码后，会将目标服务从可用服务实例列表中剔除，也就是说进行熔断，不再请求目标服务。当通过HTTP请求访问服务，返回码为502、503或504时，Istio 会将本次网络请求判断为发生错误。该属性配置的默认值是5，也就是说如果目标实例连续5个 http请求都返回了 5xx 的错误码，则该服务实例会被剔除，不再接受客户端的网络请求。
- Interval 表示服务剔除的时间间隔，即在interval 时间周期内发生1个consecutiveErrors错误，则触发服务熔断。其单位包括小时、分钟、秒和毫秒，默认值是10秒。
- baseEjectionTime 表示目标服务被剔除后，至少要维持剔除状态多长时间。这段时间内，目标服务将保持拒绝访问状态。该时间会随着被剔除次数的增加而自动增加，时间为baseEjectionTime 和驱逐次数的乘积。其单位包括小时、分钟、秒和毫秒，默认是30秒。
- maxEjectionPercent 表示可用服务实例列表中实例被移除的最大百分比，默认是10%。当超出这个比率时，即使再次发生熔断，也不会将服务剔除，这样就避免了因为瞬时错误导致大多数服务实例都被剔除的问题。
- minHealthPercent 表示健康模式的最小百分比，也就是所有服务实例中健康（未被剔除）的比率。当低于这一比率时，整个集群被认为处于非健康状态，outlierDetection配置相关的服务剔除熔断机制将被关闭，不再进行服务健康检查，所有服务实例都可以被请求访问，包含健康和不健康的主机。该属性的默认值是50%，并且minHealthPercent 和 maxEjectionPercent 的和一般都不超过100%。

### 27 | 负载均衡如何提高系统可用性？

#### 负载均衡概念

> 负载均衡往往有多种实现方式和分类标准，最为常见的分类方式为软件负载均衡和硬件负载均衡，其中软件负载均衡又分为客户端负载均衡和服务端负载均衡。

> 软件负载均衡，顾名思义，就是使用独立的负载均衡软件来实现请求的分发。它配置较为简单并且使用成本不高，能够满足大多数的负载均衡要求。但是软件所部署服务器的性能会成为整个系统吞吐量的瓶颈，并且还会产生单点错误。

> 硬件负载均衡则是依赖于特殊的负载均衡硬件设备来分发请求到不同的服务实例。相比于软件负载均衡，它能够提供更高的性能、更加多样化的负载均衡策略和更加细粒度的流量管理，更好地满足整体系统所需的负载均衡要求，但是成本极高，需要专门的硬件设备，整体投入费用较高。

> 主流的硬件负载均衡有 F5 负载均衡器等，而较为常见的软件负载均衡软件有 Nginx 和 LVS等。此外，基于 DNS 负载均衡和反向代理负载均衡也是较为主流的软件负载均衡方案。DNS负载均衡方案需要为相同的域名地址配置多个不同服务器的 IP 地址，客户端解析 DNS 地址时会根据负载均衡返回不同的 IP 地址，从而达到将请求负载到不同服务器的目的；而反向代理负载均衡使用代理服务器，客户端向同一IP地址发送请求，代理服务器将请求按照一定的规则分发到下游的服务器集群进行处理，最常见的方式即服务网关。

> 在软件负载均衡类别中，大家最为熟悉、最为常见的负载均衡方案还是反向代理负载均衡，几乎所有的主流Web服务器都支持基于反向代理的负载均衡。Web服务器实现反向代理负载均衡的核心机制就是转发HTTP请求到不同的服务实例。

> 上述方案都是服务端负载均衡，除此之外，还有客户端负载均衡，比如 Spring Cloud 的 Ribbon 组件。客户端负载均衡和服务端负载均衡的核心差异在于谁感知可用服务列表，进行负载均衡操作，客户端进行上述操作的就是客户端负载均衡，反之则是服务端负载均衡。

> 在客户端负载均衡中，客户端实例都有一份自己要访问的可用服务端地址列表数据，这些列表可从服务注册与发现中心获取，然后根据负载均衡算法选择一个实例，使用其 IP 地址来发送请求。而在服务端负载均衡中，可用服务端地址列表数据只存在于负载均衡集群中，客户端往往只能通过固定IP地址来进行调用。二者的区别如下图所示：

![服务端和客户端负载均衡对比示意图](../../Public/images/Ciqc1F9wVHqAbgVGAADqjKkTPpk445.png)

> 因为客户端负载均衡的计算执行过程是由客户端完成的，所以减轻了服务端的计算压力，并且可以根据客户端和服务端的地理或网络区域等因素进行更加优化的算法选择。但是因为客户端只是缓存了可用服务实例列表，更新并不实时，所以导致客户端可能会访问到已经宕机或者不可用的服务实例，影响用户的正常使用。而服务端负载均衡则可以更加敏锐地应对服务实例的上线、下线或变更，提供更高的稳定性和可用性。

#### 负载均衡算法简介

> 负载均衡算法定义了如何将请求分散到服务实例的规则，优秀的负载均衡算法能够有效提高系统的吞吐量，使服务集群中各服务的负载处于高效稳定的状态。常见的负载均衡算法有以下四种。

1. 随机法。该算法是随机从可用服务列表中选取一个服务实例来分发请求。它的实现非常简单，一定程度上保证了请求的分散性，但是无法顾及请求分配是否与服务实例的负载能力相符合，并且存在偶发的突然分发大量请求到同一服务实例的毛刺问题。
2. 轮询法或者加权轮询法。该算法将请求轮流分配给现有可用服务列表中的每一个服务实例，适用于集群中服务实例的负载能力大致相同且请求处理能力差异不大的场景。而改进的加权轮询则会根据各个服务实例的权重，额外分配给权重较大者相适应的更多请求，例如服务A权重为1，服务器B的权重为2，服务器C的权重为3，则使用加权轮询算法进行负载分配的过程可能为A-B-B-C-C-C-A-B-B-C-C-C。
3. Hash 法或者一致性 Hash 法。该算法根据请求的某些属性（比如说userId），使用Hash算法将其分散到不同服务实例中，这样保证了相同属性的请求会被转发到相同的服务实例中，可以更好地利用缓存，提高系统的整体性能。改进型的一致性Hash法则基于虚拟节点，在某一个服务实例宕机或不可用后能将请求平摊到其他服务节点，避免请求分发的目标实例发生剧烈的变化，影响系统的整体处理性能。
4. 最小连接数法。该算法将请求分配到当前可用服务列表中正在处理最少请求的服务实例上。该算法需要负载均衡服务器和各个服务实例之间进行一定量的信息交互，负载均衡服务器还需要了解集群中各个服务的负载情况，这样可以动态地根据服务的负载数据，更好地调控分配比例，提升系统整体性能。

#### 负载均衡算法实现

##### 1. 完全随机算法

```go
// 随机负载均衡
func (loadBalance RandomLoadBalance) SelectService(services []common.ServiceInstance) (*common.ServiceInstance, error) {
  if services == nil || len(services) == 0 {
      return nil, errors.New("service instances are not exist")
  }
  return services[rand.Intn(len(services))], nil
}
```

##### 2. 带权重的平滑轮询算法

> 带权重的平滑轮询算法是前面介绍的带权重的轮询算法的优化版本，它会根据各个服务实例的权重比例，将请求平滑地分配到各个服务实例中。而不是像权重轮询算法那样，会连续地将请求分发给相同的服务实例。

> 带权重的平滑轮询算法会根据服务实例结构体（ServiceInstance）中的权重值Weight和当前权重值CurWeight这两个属性值进行计算，这两个权重的具体定义是这样的：

- Weight是配置的服务实例权重，固定不变；
- CurWeight是服务实例目前的动态权重，一开始为0，之后会动态调整。

> 每次当请求到来，选取服务实例时，该策略会遍历服务实例队列中的所有服务实例。对于每个服务实例，让它的CurWeight 值加上 Weight 值；同时累加所有服务实例的Weight 值，将其保存为Total。

> 遍历完所有服务实例之后，如果某个服务实例的CurWeight最大，就选择这个服务实例处理本次请求，最后把该服务实例的 CurWeight 减去 Total 值。其具体实现如下所示：

```go
// 权重平滑负载均衡
func (loadBalance WeightRoundRobinLoadBalance) SelectService(services []common.ServiceInstance) (best *common.ServiceInstance, err error) {
  if services == nil || len(services) == 0 {
      return nil, errors.New("service instances are not exist")
  }
  total := 0
  for i := 0; i < len(services); i++ {
      w := services[i]
      if w == nil {
          continue
      }
      w.CurWeight += w.Weight
      total += w.Weight
      if best == nil || w.CurWeight > best.CurWeight {
          best = w
      }
  }
  if best == nil {
      return nil, nil
  }
  best.CurWeight -= total
  return best, nil
}
```

### 28 | 案例：如何在 Go 微服务中实现负载均衡？

#### 基于服务发现和注册的负载均衡

> 商品系统的负载均衡机制需要基于服务注册与发现机制，动态获取评论系统的可用实例列表，而不是将其固化在代码或者配置文件中。

#### 服务初始化

> 一致性负载均衡策略，根据商品 ID 将不同的获取商品评价的 HTTP 请求分发到某一个固定的评级服务实例上，这样有利于使用本地缓存等缓存机制，提高系统的性能。

> 一致性哈希负载均衡的核心思想是首先将服务器 key 进行 Hash 运算，将其映射到一个圆形的哈希环上，key 计算出来的整数值则为该服务实例在哈希环上的位置，然后再将请求的 key 值，用同样的方法计算出哈希环上的位置，按顺时针方向，找到第一个大于或等于该哈希环位置的服务实例 key，从而得到本次请求需要分配的服务实例。

![一致性哈希负载均衡示意图](../../Public/images/CgqCHl9wY26AWGaHAAB21ndPVIY647.png)

> 一致性哈希负载均衡策略能够很好地应对服务实例上线或者下线的场景，以防止大量请求被负载转发到不同的服务实例，减少其对整体系统带来的影响，而一般的哈希负载均衡策略就很难满足这点。比如说服务实例 node2 突然宕机下线，按照该算法，只有 Hash 值落在在服务实例 node1 和 node2 之间的请求受到了影响，被负载转发到了服务实例 node4 上，其他的大部分请求不受影响。

#### 发起网络请求

> 每次发起查询商品评论信息的网络请求前，都会先调用服务注册和发现客户端的 DiscoverServices 方法来获取当前 comment 可用的服务实例列表，然后调用负载均衡器的 SelectService 方法，根据商品的 ID 从可用列表中选中一个服务实例，最后根据该服务实例的信息构建网络请求，比如 host 和 port 信息等。整个过程如下图所示：

![基于服务发现和注册的负载均衡示意图](../../Public/images/Ciqc1F9wY4WAGxXHAABbFi54b6c310.png)

### 29 | 统一认证与授权如何保障服务安全？

> 目前主流的统一认证和授权方式有 OAuth2、分布式 Session 和 JWT 等，其中又以 OAuth2 方案使用最为广泛，已经成为当前授权的行业标准。

#### 当前行业授权标准 OAuth2

> OAuth2 是当前授权的行业标准，其重点在于为 Web 应用程序、桌面应用程序、移动设备以及室内设备的授权流程提供简单的客户端开发方式。它为第三方应用提供对 HTTP 服务的有限访问，既可以是资源拥有者通过授权允许第三方应用获取 HTTP 服务，也可以是第三方应用以自己的名义获取访问权限。

##### 1. 角色

> OAuth2 中主要分为了 4 种角色，如下表所示：

|         角色         |  中文名称  |                                      说明                                     |
|----------------------|------------|-------------------------------------------------------------------------------|
| resource owner       | 资源所有者 | 是能够对受保护的资源授予访问权限的实体。可以是一个用户，这时被称为 end-user。 |
| client               | 客户端     | 持有资源所有者的授权，代表资源所有者对受保护资源进行访问。                    |
| resource server      | 资源服务器 | 持有受保护的资源，允许持有访问令牌的请求访问受保护资源。                      |
| authorization server | 授权服务器 | 对资源所有者的授权进行认证，成功后向客户端发送访问令牌。                      |

> 在多数情况下，资源服务器和授权服务器是合二为一的：在授权交互时是授权服务器，在请求资源交互时是资源服务器。当授权服务器是单独的实体时，它可以发出被多个资源服务器接受的访问令牌。

##### 2. 协议流程

> 我们来看一张 OAuth2 的流程图，如下：

```
   +--------+                               +---------------+
   |        |--(1)- Authorization Request ->|   Resource    |
   |        |                               |     Owner     |
   |        |<-(2)-- Authorization Grant ---|               |
   |        |                               +---------------+
   |        |
   |        |                               +---------------+
   |        |--(3)-- Authorization Grant -->| Authorization |
   | Client |                               |     Server    |
   |        |<-(4)----- Access Token -------|               |
   |        |                               +---------------+
   |        |
   |        |                               +---------------+
   |        |--(5)----- Access Token ------>|    Resource   |
   |        |                               |     Server    |
   |        |<-(6)--- Protected Resource ---|               |
   +--------+                               +---------------+
```

##### 3. 客户端授权类型

> 客户端只有在获取到资源所有者的授权许可后，才能向授权服务器请求访问令牌。OAuth2 默认定义了 4 种授权类型，当然也提供了用于定义额外的授权类型的扩展机制。默认的 4 种授权类型如下表所示：

![客户端授权类型](../../Public/images/CgqCHl9_zxuAKzwlAACE5_eVSis959.png)

> 其中经常使用的授权类型为授权码类型和密码类型。简化类型是由于省略了授权码类型流程中的“授权码”步骤而得名；而客户端类型是客户端以自己的名义直接向授权服务器请求访问令牌，不需要用户授权即可请求访问令牌。

（1）授权码类型

> 授权码类型是 OAuth2 默认授权类型中功能最完整、流程最严密的授权类型。授权码类型要求客户端能够与资源所有者的代理（如 Web 浏览器等）进行交互，它通过重定向资源所有者的代理，让资源所有者与授权服务器直接交互授权，避免资源所有者的信息被泄漏，并将授权通过后生成的授权码以重定向的方式返回给客户端。

> 其授权流程图如下图所示：

![授权码类型流程图](../../Public/images/CgqCHl9_z1qAO1lbAACN7fUBGB4934.png)

> 结合该流程图，我们来分析一下授权码类型的整个工作流程。

- ①客户端将资源所有者的代理重定向到授权服务器的端点，客户端会在重定向的地址中提交自身的客户端标识、请求范围、本地状态和用于接收授权码的重定向地址等信息。
- ②资源所有者通过代理与授权服务器直接交互，授权服务器认证资源所有者的身份，并确认资源所有者同意还是拒绝访问授权。
- ③在资源所有者同意授予客户端访问权限后，授权服务器会回调客户端在第一步中提交的重定向地址，并在重定向地址中携带生成的授权码和原先提交的本地状态。否则直接返回资源所有者拒绝授权。
- ④获取到授权码的客户端可以携带授权码和用于获取授权码的重定向地址，向授权服务器请求访问令牌。授权服务器会对客户端身份和授权码同时进行认证。
- ⑤授权服务器认证客户端身份和授权码，并对客户端提交的重定向地址和获取授权码的重定向地址进行匹配。如果信息一致，则返回访问令牌，并有可能同时返回刷新令牌。

（2）密码类型

> 在密码类型中，资源所有者会将自身的密码凭证直接交予客户端，客户端通过自己持有的信息直接从授权服务器获取授权。在这种情况下，需要资源所有者对客户端高度信任，同时客户端不允许保存密码凭证。这种授权类型适用于能够获取资源所有者凭证（如用户名和密码）的客户端。授权流程图如下所示：

![密码类型授权流程图](../../Public/images/CgqCHl9_z2aANQVaAACfSJf6Wpc520.png)

（3）刷新令牌

> 以上两种类型中，你可能也注意到了，响应结果中可能会同时返回刷新令牌。那什么是刷新令牌呢？刷新令牌是授权服务器提供给客户端在访问令牌失效时重新向授权服务器申请访问令牌的凭证。

> 客户端从授权服务器中获取的访问令牌一般是具备时效性的，在访问令牌过期的情况下，持有有效用户凭证的客户端可以再次向授权服务器请求访问令牌，而持有刷新令牌的客户端也可以向授权服务器请求新的访问令牌，也就是令牌刷新操作。

#### 数据共享的分布式 Session

##### 1. 会话跟踪技术 Session 和 Cookie

> 会话是指用户登录网站后的一系列操作，比如查看列表、收藏商品和购买商品等。一次会话中一般会存在多次的 HTTP 请求。而 HTTP 协议作为一种无状态协议，在连接关闭之后，服务器就无法继续跟踪用户的会话，从而丢失了用户操作的上下文信息。

> 对此，我们需要会话跟踪技术管理和跟踪用户的整个会话，在多次 HTTP 操作中将用户与用户关联起来。而Session 和 Cookie 就是最常用的会话跟踪技术。

> Session 和 Cookie 是一种记录用户状态信息的机制，它们分别被保存在服务器端和客户端浏览器中。当客户端浏览器访问服务器的时候，服务器会把当前的用户信息以某种形式记录在服务器上，这就是 Session。客户端浏览器在访问时可以通过 Session 查找该用户的状态。

> Cookie 实际上是在客户端浏览器请求服务器时，如果服务器需要记录当前用户的状态，就会在响应中向客户端浏览器颁发一小段的文本信息用于标记当前的用户状态，这段文本信息与服务器中的 Session 一一对应，被称为 Cookie。当浏览器再次请求该网站时，会把请求的网址连同该 Cookie 提交给服务器。服务器根据 Cookie 中的信息查找 Session，从 Session 中获取用户信息，以此来辨认用户状态。服务器还可以根据需要修改 Cookie 中的内容。

> 简单来说，Cookie 被用在客户端中记录用户身份信息，而 Session 被用在服务器端中记录用户身份信息。

##### 2. 分布式 Session 的作用

> 在单体应用时代，应用部署在同一个 Web 服务器上，可以使用同一个 Web 服务器对 Session 进行管理。随着系统架构的演进，在分布式架构或者微服务架构中，会存在多个 Web 服务器，用户的请求根据负载均衡转发到不同的机器上，这就有可能导致 Session 丢失的情况出现。

> 比如，一开始用户在 A 机器上登录并发起请求，后来由于负载均衡请求被转发到 B 机器上，那这时会出现什么问题呢？因为用户的 Session 保存在 A 机器的 Web 服务器上，在 B 机器的 Web 服务器上是无法查找到的，所以导致 B 机器认为用户没有登录，返回了用户未登录的异常，引起了用户的费解。

> 因此，在分布式架构或微服务架构下，就需要保证在一个 Web 服务器上保存 Session 后，其他 Web 服务器可以同步或共享这个 Session，达到用户一次登录、多处可访问的效果。这就是分布式 Session 要做的事。

##### 3. 分布式 Session 的实现方案

> 分布式 Session 有如下几种实现方式，如下表所示：

![分布式 Session 的实现方案](../../Public/images/CgqCHl9_zyaAXi9uAAMQlnqNtDg368.png)

> 综合对比这 4 种方式，相对来说，集中式管理更加可靠，也是应用最广泛的。

#### 安全传输对象 JWT

> JWT（JSON Web Token）作为一个开放的标准，通过紧凑（快速传输，体积小）并且自包含（有效负载中包含用户所需的所有信息，避免了对数据库的多次查询）的方式，定义了用于在各方之间发送的安全 JSON 对象。

> JWT 可以很好地充当 OAuth2 的访问令牌和刷新令牌的载体，这是 Web 双方之间进行安全传输信息的良好方式。当只有授权服务器持有签发和验证 JWT 的 secret 时，也就只有授权服务器能验证 JWT 的有效性以及签发带有签名的 JWT，这就保证了以 JWT 为载体的 Token 的有效性和安全性。

> JWT 格式一般如下：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoiY2FuZyB3dSIsImV4cCI6MTUxODA1MTE1NywidXNlcklkIjoiMTIzNDU2In0.IV4XZ0y0nMpmMX9orv0gqsEMOxXXNQOE680CKkkPQcs
```

> 它由 3 部分组成，每部分通过“.”分隔开，分别是：Header（头部）、Payload（有效负载）和Signature（签名）。

##### 1. Header（头部）

> 头部通常由两部分组成。

- typ：类型，一般为 JWT。
- alg：加密算法，通常是 HMAC SHA256 或者 RSA。

> 一个简单的头部例子如下：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

> 这部分 JSON 数据会使用 Base64Url 编码后，用于构成 JWT 的第一部分，如下所示：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

##### 2. Playload（有效负载）

> 有效负载是 JWT 的第二部分，是用来携带有效信息的载体，主要是关于用户实体和附加元数据的声明，由以下 3 部分组成。

- Registered claims：注册声明。它是 JWT 预定的声明，但通常不要求强制使用。主要包含 iss（JWT 签发者）、exp（JWT 过期时间）、sub（JWT 面向的用户）、aud（接受 JWT 的一方）等属性信息。
- Public claims：公开声明。在公开声明中可以添加任何信息，一般是用户信息或者业务扩展信息等。
- Private claims：私有声明。它是由 JWT 提供者和消费者共同定义的声明，既不属于注册声明也不属于公开声明。

> Base64 对称解密的方式很容易使得加密信息被还原，所以一般不建议在 Payload 中添加任何的敏感信息。

> 一个简单的有效负载例子，如下：

```json
{

  "sub": "1234567890",
  "name": "xuan",
  "exp": 1518051157
}
```

> 这部分 JSON 会使用 Base64Url 编码后，用于构成 JWT 的第二部分，如下所示：

```
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Inh1YW4iLCJleHAiOjE1MTgwNTExNTd9
```

##### 3. Signature（签名）

> 要创建签名，就必须要有被编码后的头部、被编码后的有效负载以及一个 secret，最后通过在头部定义的加密算法 alg 加密生成签名。生成签名的伪代码如下：

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

> 上述伪代码中使用的加密算法为 HMACSHA256。Secret 作为签发密钥，用于验证 JWT 以及签发 JWT，所以只能由服务端持有，不该泄漏出去。

> 一个简单的签名如下，这就是 JWT 的第三部分。

```
X36pDQoYydHv7KDCiltTBKcQbt-iIT-jFgmUjkTSCxE
```

> 如上所述的三个部分通过“.”分割，就组成最终的 JWT。

### 30 | 如何设计基于 OAuth2 和 JWT 的认证与授权服务体系?

#### RBAC 和 ACL

> 认证和授权本质上是对用户的访问权限进行控制，关于权限系统的设计就不得不提及经典的RBAC（Role-Based Access Control）模型，即基于角色的访问控制模型。

> RBAC 模型将用户的操作过程抽象为 Who、What 和 How 这三者之间的关系，换句话说，就是 Who 可以对 What 进行 How 的操作。RBAC 模型将用户通过角色与权限进行关联，用户通过成为某一角色而获得该角色的权限；一个用户可以有多个角色，而每个角色又拥有多种权限。通过“用户-角色-权限”的授权模型，大大简化了权限的管理。

> 根据 LIST 提出的 RBAC96 模型，RBAC 由 4 个概念性模型组成，分别为：

- RBAC0，作为基本模型，定义了完全实现 RBAC 模型的系统的最低需求，它是 RBAC 的核心所在；
- RBAC1，在包含 RBAC0 的基础上，增加了角色分层的概念，一个角色可以从另一个角色中继承权限；
- RBAC2，同样在 RBAC0 的基础上，引入静态职责分离和动态职责分离，它是 RBAC 的约束模型；
- RBAC3，作为统一模型，在基于 RBAC0 的基础上，整合了 RBAC1 和 RBAC2。

> RBAC96 模型族关系图如下所示：

![RBAC96 模型族关系图](../../Public/images/CgqCHl-BjmSASE66AAAcST7W53k950.png)

> 除了 RBAC 外，还有一种比较流行的权限设计方案——ACL（Access Control Lists）模型，即访问控制列表。ACL 的核心思想是将用户直接和权限关联，这样就可以非常简单地完成访问控制。但是在权限过多时，却又很容易造成授予权限时的复杂性，访问控制列表的管理最终会演变成极为烦琐的工作。

> 为了简化权限设计方案，便于实践的开展，我们就基于 RBAC0 模型描述的用户、角色、权限和会话关系，设计如下的权限模型：

![简易角色权限模型图](../../Public/images/Ciqc1F-BjoeAOEb7AAC052s4Czk102.png)

> 上图中分别有用户、角色和权限 3 个实体，每个用户可以拥有多种角色，每一种角色都是一定数量权限的集合，它们之间通过关联表关联起来。当要授予某个用户部分权限时，可以将对应的角色赋予用户，比如在一个论坛系统中，帖子管理员具备审核帖子、删除帖子等权限，当有新的用户希望管理帖子时，可以将帖子管理员的角色授予该用户，这样他就具备了帖子管理的权限。

#### OAuth2 和 JWT

> 在微服务架构中，会有多种多样的客户端通过网关接入系统中请求服务，我们可以对这些客户端进行简单的分类，包括第三方客户端应用和自家客户端应用两类。

> 第三方客户端应用基本为外部系统，希望请求本系统内的资源或者服务，为了避免将系统内的用户信息泄漏给第三方客户端，系统将通过授权码类型给这类客户端提供访问令牌。而对于自家客户端应用，因为是可信任的客户端，所以可以允许它们通过密码类型获取访问令牌，即它们能够直接接触和保存用户的密码凭证等信息。

![多客户端访问模型图](../../Public/images/CgqCHl-BjpWAOslFAAA3GP2cSQQ804.png)

> 我们可以将 JWT 用作访问令牌和刷新令牌的载体，将签发 JWT 的 secret 保存在授权服务器，由授权服务器签发 JWT 样式的访问令牌。同时，我们还可以将用户的相关信息，比如用户 ID、用户角色码和权限码等信息放到有效负载中，当下游资源服务器使用用户的信息时，可以直接从访问令牌中解析获取，而无须重复查询数据库，这就减少了请求消耗，提升了访问效率。

![携带用户信息的 JWT 样式访问令牌](../../Public/images/CgqCHl-BjqKAJH-jAAAqGAJDXv8651.png)

#### Token 中继

> 在微服务架构中，各个微服务之间存在错综复杂的调用关系，而且在调用过程中很可能需要获取用户的身份信息，甚至部分接口还需要对用户拥有的权限进行鉴权。虽然我们可以在业务请求体中要求上游服务携带用户的相关信息，然后在业务实现中根据用户信息，去授权服务器查找用户对应的权限信息并进行鉴权操作，但是这种方式对业务开发人员不友好，需要他们在业务实现代码中频繁传递用户信息。同时这其中的部分通用鉴权行为，完全可以通过资源服务器的方式实现，从而减轻业务开发工作。

> 我们可以将访问令牌在多个请求中进行传递，也就是Token 中继，由资源服务器根据自身的认证和鉴权配置，对访问令牌中用户身份和权限进行鉴权，如下图所示：

![在资源服务器中传递访问令牌](../../Public/images/Ciqc1F-BjrSAFuiKAACSXWInd6I057.png)

> 如果访问令牌是 JWT 样式，并且其中包含了用户身份信息和权限信息，那么资源服务器就可以直接根据其内的信息进行鉴权操作，从而避免频繁向权限系统请求用户的角色和权限信息。同时，资源服务器应该具备 Token 中继的能力，自动将请求中的访问令牌携带到下游，让鉴权操作对业务开发人员透明。

#### 整体结构设计

> 基于上面的介绍，我们可以将微服务统一认证与授权服务体系设计为如下：

![统一认证与授权服务整体结构设计](../../Public/images/Ciqc1F-Bjr2AVEhFAABGPArWoOo510.png)

> 上述这个设计图的整体结构可描述为如下：

- 客户端作为用户的代理，通过授权码类型或者密码类型向授权服务器请求访问令牌。客户端获取到对应的访问令牌后，即可通过网关请求对应的微服务访问用户存储在系统中的资源。
- 网关服务不仅仅起到为下游微服务做反向代理的作用，还作为资源服务器去验证请求携带的访问令牌的有效性。JWT 样式的访问令牌在颁发之后，在 JWT 注册声明的有效时间到达之前，访问令牌都会被认为是有效的。考虑到这个问题，我们就需要在授权服务器中维护一个用户与有效访问令牌的映射，在令牌主动失效时，将该条映射关系主动删除，以保证失效后的访问令牌不可用。因此，网关在检验访问令牌的有效性时，是需要请求授权服务器进行验证，验证不通过的请求将被直接拒绝。
- 授权服务器持有签发 JWT 样式访问令牌的 sercet，在请求访问令牌时，它会验证客户端的身份和携带的用户凭证或者授权码信息，给通过验证的客户端颁发合法的访问令牌。同时，授权服务器会接收网关关于令牌验证的请求，通过合法的访问令牌的请求，拒绝非法请求。除此之外，授权服务器还具备权限管理的能力。
- 业务服务在处理业务时，可能需要获取当前访问用户的身份信息，在进行敏感操作时还会对用户的角色和权限进行鉴权。我们可以通过 Token 中继机制，在多个资源服务器之间传递访问令牌。由于访问令牌为 JWT 样式，资源服务器可以直接从访问令牌中解析出用户相关的角色和权限信息进行鉴权。

### 31 | 案例：如何自定义授权服务器？

> 授权服务器的主要交互对象为客户端和资源服务器，它们之间的交互流程如下图所示：

![交互流程示意图](../../Public/images/Ciqc1F-GurGAAcYMAADV-kU9nYM616.png)

> 从这个交互流程我们可以发现，授权服务器的主要职责是颁发令牌和验证访问令牌。对此，授权服务器需要对外提供两个相应的接口：

- /oauth/token 用于客户端携带用户凭证请求访问令牌；
- /oauth/check_token 用于验证访问令牌的有效性，返回访问令牌绑定的客户端和用户信息。

> 除此之外，我们还可以让授权服务器承担客户端信息管理和权限管理等额外功能，也可以由其他的服务提供该部分能力。每个客户端都可以为用户申请访问令牌，访问令牌是与申请的客户端、授权的用户绑定的，表示某一用户授予某一个客户端有限的访问资源权限。

> 一个基本的授权服务器应该由以下五个模块组成，如图所示：

![授权服务器主要模块组成图](../../Public/images/Ciqc1F-GuwaAGy2gAABuKAtfMbk448.png)

- TokenGranter（令牌生成器），根据客户端请求的授权类型，使用不同的方式验证客户端和用户信息，并使用 TokenService 生成访问令牌；
- TokenService（令牌服务），生成并管理令牌，它使用 TokenStore 存储令牌；
- TokenStore（令牌存储器），负责令牌的存取工作；
- ClientDetailsService（客户端详情服务），根据 ClientId 查询系统中注册的客户端信息；
- UserDetailsService（用户详情服务），用于获取用户信息。

### 32 | 案例：如何保证微服务实例资源安全？

#### 整体结构

> 资源服务器会在请求进入具体的资源端点之前，对请求中携带的访问令牌进行校验，比较常用的做法是采用拦截器的方式实现，如下图所示:

![资源服务器中请求流程图](../../Public/images/Ciqc1F-JYciAXOBBAACNYK3Blio069.png)

> 请求在进入具体的资源端点之前，会至少经过令牌认证拦截器和权限检查拦截器这两个拦截器，以及其他发挥重要功能的拦截器，比如限流拦截器等。令牌认证拦截器会解析请求中携带的访问令牌，请求授权服务器验证访问令牌的有效性，明确当前请求的客户端和用户信息，并把这些信息写入请求上下文中，如果访问令牌无效，将会拒绝请求，返回认证错误。权限检查拦截器会按照预设的权限规则对请求上下文中的客户端和用户信息进行权限检查，如果权限不足也会拒绝访问，返回鉴权错误。

> 对此我们可以将资源服务器设计为以下几个模块，如图所示：

![资源服务器模块组成图](../../Public/images/CgqCHl-JYdCAXd-5AACLyd4sStg095.png)

- OAuth2AuthorizationContext（认证上下文处理器），负责从请求解析出访问令牌，委托 ResourceServerTokenService 验证访问令牌的有效性，获取令牌对应的客户端和用户信息。
- OAuth2AuthorizationMiddleware（认证中间件），检查请求上下文是否存在客户端和用户信息。
- AuthorityAuthorizationMiddleware（权限检查中间件），从请求上下文中获取客户端和用户信息，并根据预设的权限规则对请求的客户端和用户信息进行鉴权。
- ResourceServerTokenService（资源服务器令牌服务），帮助资源服务器检验访问令牌的有效性以及获取访问令牌绑定的客户端和用户信息。

### 33 | 如何追踪分布式系统调用链路的问题？

> 传统方式

- 通过打日志的方式来进行埋点
- 直接连接服务器进行代码调试

#### 为什么需要分布式链路追踪

> 分布式链路追踪不仅能够帮助开发者直观分析请求链路，快速定位性能瓶颈，逐渐优化服务间的依赖，而且还有助于开发者从宏观角度更好地理解整个分布式系统。

#### 什么是分布式链路追踪

> 所谓分布式链路追踪，就是记录一次分布式请求的调用链路，并将分布式请求的调用情况集中展示。其中，调用详情包括各个请求的服务实例信息、服务节点的耗时、每个服务节点的请求状态等；分布式链路追踪还可以分析出请求的依赖拓扑，即这次请求涉及哪些服务、这些服务上下游的关系等，这对于排查性能瓶颈非常有帮助。

#### 链路追踪与日志、Metrics 的关系

> Tracing 表示链路追踪，Logging 和 Metrics 是与之相近的两个概念。这三者的关系如下图所示：

![Tracing & Logging & Metrics三者的关系](../../Public/images/Ciqc1F-GvOSAV_BRAAEKN28KEAQ070.png)

- Tracing：记录单个请求的处理流程，其中包括服务调用和处理时长等信息。
- Logging：用于记录离散的日志事件，包含程序执行到某一点或某一阶段的详细信息。
- Metrics：可聚合的数据，通常是固定类型的时序数据，包括 Counter、Gauge、Histogram 等。

> 同时，这三者相交的情况（或者说混合出现）也比较常见。

- Logging & Metrics：可聚合的事件。例如，分析某对象存储的 Nginx 日志，统计某段时间内 GET、PUT、DELETE、OPTIONS 操作的总数。
- Metrics & Tracing：单个请求中的可计量数据。例如，SQL 执行总时长、gRPC 调用总次数等。
- Tracing & Logging：请求阶段的标签数据。例如，在 Tracing 的信息中标记详细的错误原因。

> 针对这每种分析需求，我们都有非常强大的集中式分析工具。

- Logging：ELK（Elasticsearch、Logstash和Kibana），Elastic 公司提供的一套完整的日志收集以及展示的解决方案。
- Metrics：Prometheus，专业的Metric 统计系统，存储的是时序数据，即按相同时序（相同名称和标签），以时间维度存储连续数据的集合。
- Tracing：Jaeger，是 Uber 开源的一个兼容 OpenTracing 标准的分布式追踪服务。

> 通过以上讲解，你现在应该知道，Tracing、Logging 和 Metrics 这三者之间有一定的关系，既可以单独使用，也可以组合使用。每一个组件都有其侧重点，Tracing 用于追踪具体的请求，绘制调用的拓扑；Logging 则是主动记录的日志事件；Metrics 记录了请求相关的时序数据，通常用于性能统计。在分布式系统中，这三者通常是组合在一起使用。

#### 分布式链路追踪的基础概念

> 分布式链路追踪组件涉及 Span、Trace、Annotation 等基本概念，这些概念还是比较重要的，所以下面我们就具体介绍下这些概念。

- Span，分布式链路追踪组件的基本工作单元。一次请求，即发起的一次链路调用（可以是 RPC、DB 调用等）会创建一个 Span。通过一个64位ID标识Span，通常使用 UUID，Span 中还有其他的数据，例如描述信息、时间戳、parentID 等，其中 parentID 用来表示 Span 调用链路的层级关系。
- Trace，Span 集合，类似树结构。表示一条完整的调用链路，存在唯一标识。Trace 代表了一个事务或者流程在系统中的执行过程，由多个 Span 组成的一个有向无环图，每一个 Span 代表 Trace 中被命名并计时的连续性的执行片段。
- Annotation，注解。用来记录请求特定事件相关信息（例如时间），通常包含 4 种注解信息，分别包括：①CS（Client Sent），表示客户端发起请求；②SR（Server Received），表示服务端收到请求；③SS（ServerSent），表示服务端完成处理，并将结果发送给了客户端；④CR（Client Received），表示客户端获取到服务端返回信息。

> 链路信息的还原依赖于两种数据：一种是各个节点产生的事件，如 CS、SS，称为带外数据，这些数据可以由节点独立生成，并且需要集中上报到存储端；另一种数据是TraceID、SpanID、ParentID，用来标识 Trace、Span 以及 Span 在一个Trace中的位置，称为带内数据，这些数据需要从链路的起点一直传递到终点。

> 通过带内数据的传递，可以将一个链路的所有过程串起来；通过带外数据，可以在存储端分析更多链路的细节。

> 下图展示了 Trace 树的运行机制，通过 Trace 树我们可以了解一次请求过程中链路追踪的基本运行原理。

![Trace 树](../../Public/images/CgqCHl-GvPCASsTBAACuOCx60p8798.png)

> Span 表示一个服务调用的开始和结束时间，即执行的时间段。分布式链路追踪组件记录了 Span 的名称以及每个 SpanID 的 ParentID，如果一个 Span 没有 ParentID 则被称为 Root Span，当前节点的 ParentID 即为调用链路上游的 SpanID，所有的 Span 都属于一个特定的 Trace，共用一个 TraceID。

### 34 | OpenTracing 规范介绍与分布式链路追踪组件选型

#### 分布式链路追踪规范：OpenTracing

> 对于链路追踪组件来说，其核心步骤有：代码埋点、数据存储和查询展示。链路追踪组件的组成，如下图所示：

![链路追踪组件的组成](../../Public/images/CgqCHl-SjWKARLMtAACUh0mRaIk002.png)

> OpenTracing 的诞生主要是为了解决不同的分布式链路追踪平台的 API 兼容问题

> OpenTracing 是一个轻量级的标准化层，位于应用程序/类库和追踪或日志分析程序之间。它提供了多种语言库的支持，包括 Go、Java、Python、Objective-C 和 C++ 等，通过引入这些通信标准库，我们就可以将追踪的信息发送到指定的组件。

> OpenTracing 的架构如下图所示，从图中我们可以看到 OpenTracing 支持 Zipkin、LightStep 和 Appdash 等追踪组件，并且可以轻松集成到开源的框架中，例如 gRPC、Flask、Django 和 Go-kit 等。

> OpenTracing 是一个 Library 库，定义了一套通用的数据上报接口，要求各个分布式追踪系统都来实现这套接口。这样一来，应用程序只需要对接 OpenTracing，而无须关心后端采用的到底是什么样的分布式追踪系统，因此开发者可以无缝切换分布式追踪系统，也使得在通用代码库增加对分布式追踪的支持成为可能。

> OpenTracing 于 2016 年 10 月加入 CNCF 基金会，是继 Kubernetes 和 Prometheus 之后，第三个加入 CNCF 的开源项目。它是一个中立的（厂商无关、平台无关）分布式追踪的 API 规范，提供统一接口，可方便开发者在自己的服务中集成一种或多种分布式追踪的实现。

#### 流行的分布式链路追踪组件

##### 1. 简单易上手的 Zipkin

![Zipkin 架构图](../../Public/images/CgqCHl-Gvh6AVPPFAABoGNmX9as934.png)

> 从 Zipkin 架构图可知，Zipkin 包含 Collector、Storage、API 和 Web UI 这 4 个部分。

- Collector：存储和索引报上来的链路数据，以供后续查找。
- Storage：Zipkin 的服务端存储，除了 Cassandra，Zipkin 还原生支持 ElasticSearch 和 MySQL。
- Zipkin Query Service（API）：Zipkin 为 Web UI 提供的一个简单查询 API，用于查找和检索链路调用信息。
- Web UI：Zipkin 查询链路追踪的界面。Web UI 提供了一种基于服务、时间和注解查看 Trace 记录的方法。

##### 2. 云原生链路监控组件 Jaeger

> Jaeger 的架构如下图所示：

![Jaeger 架构图](../../Public/images/CgqCHl-GvjaAElw5AAEfQ3MA-FU057.png)

> 我们来分析一下 Jaeger 的架构图，Jaeger 主要由 client、agent、collector、DB 和 query 这几部分组成。

- jaeger-client，即 Jaeger 客户端，各个语言的应用程序通过客户端 API 写入数据，并将链路追踪的数据发送给 jaeger-agent。
- jaeger-agent，Jaeger 代理服务，每个物理机都会部署 agent，是一个守护进程，用于将数据发送给 Jaeger collector。jaeger-agent 是 client 和 collector 之间的桥梁，负责将二者解耦。
- jaeger-collector，负责将客户端数据写入后端存储。
- Data Store，服务端存储，目前支持 Cassandra 和 ElasticSearch 两种方式的数据存储。
- jaeger-query，用于检索链路调用信息，通过 UI 进行展示。

##### 3. SkyWalking

> SkyWalking 整体架构的模块较多，但是结构比较清晰，主要就是通过收集各种格式的数据进行存储，然后展示。如下图为 SkyWalking 6.x 的架构图：

![SkyWalking 6.x 架构图](../../Public/images/Ciqc1F-GvlOARTWZAANEwHNPMrQ811.png)

##### 4. 链路统计详细的 Pinpoint

> Pinpoint 是一个 APM 工具，适用于 Java 、PHP 编写的大型分布式系统进行链路追踪。也就是说，Go 语言项目不能直接应用 Pinpoint，需要使用代理进行改造。

![Pinpoint 链路监控页面](../../Public/images/Ciqc1F-GvmSAYLmbAAI33y2CSDk491.png)

###### 5. 组件的指标对比

![组件的指标对比](../../Public/images/Ciqc1F-SjXKAKnGuAAGS0Nd0F1o697.png)

### 35 | 案例：如何在微服务中集成 Zipkin 组件？

#### 应用架构图

![Go-kit 集成 Zipkin 的应用架构图](../../Public/images/Ciqc1F-GvryAbSsGAABD2LP4yN8428.png)

> 从架构图中可以看到：我们构建了一个服务网关，通过 API 网关调用具体的微服务，所有的服务都注册到 Consul 上；当客户端的请求到来之时，网关作为服务端的门户，会根据配置的规则，从 Consul 中获取对应服务的信息，并将请求反向代理到指定的服务实例。

### 36 | 如何使用 ELK 进行日志采集以及统一处理？

> 微服务架构中的日志收集方案 ELK（ELK 是 Elasticsearch、Logstash 和 Kibana 的简称），准确地说是 ELKB，即 ELK + Filebeat，其中 Filebeat 是用于转发和集中日志数据的轻量级传送工具。

#### ELKB 分布式日志系统

> ELKB 分别是指 Elasticsearch、Logstash、Kibana 和 Filebeat。Elastic 提供的一整套组件可以看作是 MVC 模型：Logstash 对应逻辑控制 Controller 层，Elasticsearch 是一个数据模型 Model 层，而 Kibana 则是视图 View 层。Logstash 和 Elasticsearch 是基于 Java 编写实现的，Kibana 则使用的是 Node.js 框架。

#### Logstash 的安装与使用

> Logstash 是一个数据分析软件，主要用于分析 Log 日志。其工作原理如下所示：

![Logstash 工作原理图](../../Public/images/Ciqc1F-WruaAULiSAABppBq55b8500.png)

#### Filebeat 的安装与使用

> ELKB 中的 Filebeat 是最后研发的，并剥离出 Logstash 的数据转发功能。Filebeat 基于 Go 语言开发，是用于转发和集中日志数据的轻量级传送工具。通过配置 Filebeat，我们可以监听日志文件或位置、收集日志事件，并将这些文件转发到 Logstash、Kafka、Redis 等，当然也可以直接转发到 Elasticsearch 进行索引。

![Filebeat 工作原理图](../../Public/images/CgqCHl-WrwmAbLLsAAEIOMM6eEI271.png)

#### ELKB 的使用实践

>  ELKB 收集日志的流程：

![ELKB 收集日志的流程图](../../Public/images/Ciqc1F-bhKiASToRAAC-S76rRng777.png)

### 37 | 如何处理 Go 错误异常与并发陷阱？

> Go 中主要通过 error 和 panic 分别表示错误和异常，并提供了较为简洁的错误异常处理机制。

#### Errors are values

> 在我过去接触的编程语言中，大多是通过try-catch 的方式对可能出现错误的代码块进行包装：程序运行 try 中代码，如果 try 中的代码运行出错，程序将会立即跳转到 catch 中执行异常处理逻辑。

> 与其他的编程语言不同，Go 中倡导“Errors are values!”的处理思想，它将 error 作为一个返回值，来迫使调用者对 error 进行处理或者忽略。于是，在代码中我们将会编写大量的 if 判断语句对 error 进行判断，如下所示：

> 在 Go 1.13 版本之后，errors 包中添加了 errors.Is 和 errors.As 函数：errors.Is 方法用来比较两个 error 是否相等，而 errors.As 函数用来判断 error 是否为特定类型。

> 由于 error 是一个值，因此我们可以对其进行编程，简化 Go 错误处理的重复代码。在一些管道和循环的代码中，只要其中一次处理出现错误，就应该退出本次管道或者循环。寻常的做法是在每次迭代都检查错误，但为了让管道和循环的操作显得更加自然，我们可以将 error 封装到独立的方法或者变量中返回，以避免错误处理掩盖控制流程

#### defer、panic 和 recover

> defer 是 Go 中提供的一种延迟执行机制，每次执行 defer，都会将对应的函数压入栈中。在函数返回或者 panic 异常结束时，Go 会依次从栈中取出延迟函数执行。

> defer 有三个比较重要的特点。第一个是按照调用 defer 的逆序执行，即后调用的在函数退出时先执行，后进先出。

> 第二个特点是 defer 被定义时，参数变量会被立即解析，传递参数的值拷贝。在函数内使用的变量其实是对外部变量的一个拷贝，在函数体内，对变量更改也不会影响外部变量

> 然而当 defer 以闭包的方式引用外部变量时，则会在延迟函数真正执行的时候，根据整个上下文确定当前的值

> 在日常开发中，我建议你还是不要在循环中使用 defer。因为相较于直接调用，defer 的执行存在着额外的开销，例如 defer 会对其后需要的参数进行内存拷贝，还会对 defer 结构进行压栈出栈操作。因此，在循环中使用 defer 可能会带来较大的性能开销。

> defer 的第三个特点是可以读取并修改函数的命名返回值，如下面的例子所示：

```go
func main()  {
  fmt.Println(test())
}
func test() (i int) {
  defer func() { i++ }()
  return 1
}

// 2
```

> 这是因为对于命名返回值，defer 和 return 的执行顺序如下：

- 将 1 赋给 i；
- 执行 i++；
- 返回 i 作为函数返回值。

> defer 的内部实现为一个延迟调用链表，如下图所示：

![defer 延迟调用链表示意图](../../Public/images/Ciqc1F-iaTOALrVCAAC28gcQ66k361.png)

> 其中，g 代表 goroutine 的数据结构。每个 goroutine 中都有一个 `_defer` 链表，当代码中遇到 defer 关键字时，Go 都会将 defer 相关的函数和参数封装到 `_defer` 结构体中，然后将其注册到当前 goroutine 的 `_defer` 链表的表头。在当前函数执行完毕之后，Go 会从 goroutine 的 `_defer` 链表头部取出来注册的 defer 执行并返回。

> `_defer` 结构体中存储 defer 执行相关的信息，定义如下所示：

```go
type _defer struct {
  siz       int32 // 参数与结果内存大小
  started   bool 
  heap      bool // 是否在堆上分配
  openDefer bool //是否经过开放编码优化
  sp        uintptr // 栈指针
  pc        uintptr // 调用方的程序计数器
  fn        *funcval // defer 传入的函数
  _panic    *_panic 
  link      *_defer // 下一个 _defer
}
```

> panic 是一个内置函数，用于抛出程序执行的异常。它会终止其后将要执行的代码，并依次逆序执行 panic 所在函数可能存在的 defer 函数列表；然后返回该函数的调用方，如果函数的调用方中也有 defer 函数列表，也将被逆序执行，执行结束后再返回到上一层调用方，直到返回当前 goroutine 中的所有函数为止，最后报告异常，程序崩溃退出。异常可以直接通过 panic 函数调用抛出，也可能是因为运行时错误而引发，比如访问了空指针等。

> 而recover 内置函数可用于捕获 panic，重新恢复程序正常执行流程，但是 recover 函数只有在 defer 内部使用才有效。如下面例子所示：

```go
func main() {
  err := panicAndReturnErr()
  if err != nil{
    fmt.Printf("err is %+v\n", err)
  }
  fmt.Println("returned normally from panicAndReturnErr")
}

func panicAndReturnErr() (err error){
  defer func() {
    // 从 panic 中恢复
    if e := recover(); e != nil {
      // 打印栈信息
      buf := make([]byte, 1024)
      buf = buf[:runtime.Stack(buf, false)]
      err = fmt.Errorf("[PANIC]%v\n%s\n", e, buf)
    }
  }()
  fmt.Println("panic begin")
  panic("panic this game")
  fmt.Println("panic over")
  return nil
}
```

> 预期的执行结果为：

```
panic begin
err is [PANIC]panic this game
goroutine 1 [running]:
main.panicAndReturnErr.func1(0xc000062f08)
  /Users/apple/Desktop/micro-go-course/section37/defer_example.go:21 +0xa1
panic(0x10ad640, 0x10eb360)
  /usr/local/go/src/runtime/panic.go:969 +0x166
main.panicAndReturnErr(0x0, 0x0)
  /Users/apple/Desktop/micro-go-course/section37/defer_example.go:26 +0xc2
main.main()
  /Users/apple/Desktop/micro-go-course/section37/defer_example.go:10 +0x26
returned normally from panicAndReturnErr
```

> 从这个执行结果可以看出，panicAndReturnErr 函数在 panic 之后将会执行 defer 定义的延迟函数，恢复程序的正常执行逻辑。在上述例子中，我们在 defer 函数中使用 recover 函数帮助程序从 panic 中恢复过来，并获取异常堆栈信息组成 error 返回调用方。panicAndReturnErr 从 panic 中恢复后将直接返回，不会执行函数中 panic 后的其他代码。

#### 常见的并发陷阱

> 首先是循环并发时闭包传递参数的问题，如下错误例子所示：

```go
func main()  {
  for i := 0 ; i < 5 ; i++{
    go func() {
      fmt.Println("current i is ", i)
    }()
  }
  time.Sleep(time.Second)
}
// 这段代码极有可能的输出为：
// current i is 5
// current i is 5
// current i is 5
// current i is 5
// current i is 5
```

> 这是因为 i 使用的地址空间在循环中被复用，在 goroutine 执行时，i 的值可能在被主 goroutine 修改，而此时其他 goroutine 也在读取使用，从而导致了并发错误。针对这种错误可以通过复制拷贝或者传参拷贝的方式规避，如下所示：

```go
func main()  {
  for i := 0 ; i < 5 ; i++{
    go func(v int) {
      fmt.Println("current i is", v)
    }(i)
  }
  time.Sleep(time.Second)
}
```

> 前面介绍 panic 时我们了解到 panic 异常的出现会导致 Go 程序的崩溃。但其实即使 panic 是出现在其他启动的子 goroutine 中，也会导致 Go 程序的崩溃退出，同时 panic 只能捕获 goroutine 自身的异常，因此**对于每个启动的 goroutine，都需要在入口处捕获 panic，并尝试打印堆栈信息并进行异常处理，从而避免子 goroutine 的 panic 导致整个程序的崩溃退出**。如下面的例子所示：

```go
func RecoverPanic() {
  // 从 panic 中恢复并打印栈信息
  if e := recover(); e != nil {
    buf := make([]byte, 1024)
    buf = buf[:runtime.Stack(buf, false)]
    fmt.Printf("[PANIC]%v\n%s\n", e, buf)
  }
}
func main() {
  for i:= 0 ; i < 5 ; i++{
    go func() {
      // defer 注册 panic 捕获函数
      defer RecoverPanic()
      dothing()
    }()
  }
}
```

> 最后一个技巧是要善于结合使用 select、timer 和 context 进行超时控制。在 goroutine 中进行一些耗时较长的操作，最好都加上超时 timer，在并发的时候也要传递 context，这样在取消的时候就不会有遗漏，进而达到回收 goroutine 的目的，避免内存泄漏的发生。如下面的例子所示，通过 select 同时监听任务和定时器状态，在定时器到达而任务未完成之时，提前结束任务，清理资源并返回。

```go
select {
// do logic process
case msg <- input:
   ....
// has been canceled
case <-ctx.Done():
    // ...资源清理
    return
// 2 second timeout
case <-time.After(time.Second * 2)
    // ...资源清理 
    return
default:
}
```

### 38 | 案例：如何使用 Prometheus 和 Grafana 监控预警服务集群？

> Prometheus 和 Grafana 相结合是开源服务监控和预警平台的主流方案之一。

> 建立完善的监控体系，可以达到以下四个目的：

- 数据可视化
- 跟踪和分析程序的长期指标趋势。
- 故障和异常告警。 
- 故障和异常分析与定位。

#### 部署和配置 Prometheus

> Prometheus 是一套开源的基于时间序列数据库的数据监控报警平台。

- 多维数据模型，时序列数据由 metric 名和一组 key/value 组成，方便存储不同格式和模型的数据；
- 可以使用自建的 PromQL 在多维度上灵活地进行数据查询，方便数据检索和查询；
- 不依赖分布式存储，单主节点工作，方便自身部署和运维；
- 可以采用 Push 和 PushGateway 结合的方式进行数据收集，方便特殊网络状况下的数据收集；
- 可以通过服务发现或者静态配置来获取要进行数据采集的目标服务或机器，方便动态配置数据采集的目标；
- 支持多种可视化图表及表盘，方便数据可视化。

> 此外，Prometheus 采集数据时使用拉模型（Pull），通过 HTTP 协议调用 Exporter 的接口来采集数据指标，只要服务能够提供 HTTP 接口就可以接入 Prometheus 监控系统，相较于私有协议或二进制协议来说，接入成本低且开发简易，这无疑简化了应用系统纳入该系统的工作量，并降低了难度。

