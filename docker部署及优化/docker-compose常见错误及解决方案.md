## 错误 1：频繁的容器重建

- 处理非容器化应用程序的传统工作流如下：编码 => 构建 => 运行

- 一个docker build步骤。他们的工作流如下：编码 => 构建 => 容器构建 => 运行

- ### 解决方案：

  1. #### 在 Docker 外运行你的代码

     - 在 Docker Compose 中启动所有依赖项，但在本地运行你正在积极处理的代码。这模仿了开发非容器化应用程序的工作流
     - 只需要在localhost上暴露你的依赖，并将你正在使用的服务指向localhost:<port>地址

  2. #### 最大化缓存来优化 Dockerfile

     - 生产环境的 Dockerfile 文件的典型模式是通过将单个命令链接到一个RUN语句中来减少层数

       - Dockerfile 文件可能如下所示： 每次重新运行该命令时，Docker 都会重新下载所有的依赖并重新安装它们。

         ```bash
         RUN \
             go get -d -v \
             && go install -v \
             && go build
         ```

       - 一个专门用于开发环境的的 Dockerfile 文件。将每件事都分解成非常小的步骤，规划好你的 Dockerfile 文件，这样基于经常变化的代码的步骤最后执行

       - 最不频繁更改的内容，例如拉取依赖，应该放在第一位。这样，在重建 Dockerfile 时就不必构建整个项目。你只需要构建你刚刚修改的一小部分

         ```bash
         FROM golang:1.13-alpine as builder
         
         RUN apk add busybox-static
         
         WORKDIR /go/src/github.com/kelda-inc/blimp
         
         ADD ./go.mod ./go.mod
         ADD ./go.sum ./go.sum
         ADD ./pkg ./pkg
         
         ARG COMPILE_FLAGS
         
         RUN CGO_ENABLED=0 go install -i -ldflags "${COMPILE_FLAGS}" ./pkg/...
         
         ADD ./login-proxy ./login-proxy
         RUN CGO_ENABLED=0 go install -i -ldflags "${COMPILE_FLAGS}" ./login-proxy/...
         
         ADD ./registry ./registry
         RUN CGO_ENABLED=0 go install -i -ldflags "${COMPILE_FLAGS}" ./registry/...
         
         ADD ./sandbox ./sandbox
         RUN CGO_ENABLED=0 go install -i -ldflags "${COMPILE_FLAGS}" ./sandbox/...
         
         ADD ./cluster-controller ./cluster-controller
         RUN CGO_ENABLED=0 go install -i -ldflags "${COMPILE_FLAGS}" ./cluster-controller/...
         
         RUN mkdir /gobin
         RUN cp /go/bin/cluster-controller /gobin/blimp-cluster-controller
         RUN cp /go/bin/syncthing /gobin/blimp-syncthing
         RUN cp /go/bin/init /gobin/blimp-init
         RUN cp /go/bin/sbctl /gobin/blimp-sbctl
         RUN cp /go/bin/registry /gobin/blimp-auth
         RUN cp /go/bin/vcp /gobin/blimp-vcp
         RUN cp /go/bin/login-proxy /gobin/login-proxy
         
         FROM alpine
         
         COPY --from=builder /bin/busybox.static /bin/busybox.static
         COPY --from=builder /gobin/* /bin/
         ```

         通过最近引入的多阶段构建，现在可以创建具有良好分层且镜像很小的 Dockerfile

  3. #### 使用主机卷

     - 使用主机卷来直接将代码加载到容器上，使能够以本机速度运行代码，同时在包含运行时依赖项的 Docker 容器中运行
     - 更改会自动同步到容器中，然后能立即在容器中执行

## 错误2：主机卷速度慢

- 主机卷在 Windows 和 Mac 上读写文件的速度非常慢

-  Docker 是运行在 Windows 和 Mac 的一个虚拟机上，在进行主机卷加载时，必须经过大量的转换才能将电脑的文件夹加载到容器中

-  Linux 本机上运行 Docker 时不会出现这些情况

- ### 解决方案

  1. #### 放松强一致性

     - 文件系统加载会保持强一致性，所有特定文件的读取者和写入者都同意任何文件修改发生的顺序，从而同意该文件的内容

     - 在 Docker Compose 中，你只需将cached关键词添加到卷加载中即可获得显著的性能保证

       ```bash
       volumes:
           - "./app:/usr/src/app/app:cached"
       ```

  2. #### 代码同步

     - 设置代码同步。你可以用一个工具来通知你的笔记本电脑和容器之间的更改，并复制文件来解决差异（类似于 rsync），而不是加载一个卷
     - Docker 的下一个版本内置了Mutagen，作为卷的缓存模式的一种替代
     - 也可以直接下载 Mutagen 项目

  3. #### 不要加载包

     - 对于 Node 这样的语言，大部分文件操作往往位于包目录（例如node_modules），从卷中排除这些目录会显著提高性能

     - 将代码加载到一个容器中。然后用它自己干净的专用卷覆盖了node_modules目录

       ```bash
       volumes:
         - ".:/usr/src/app"
         - "/usr/src/app/node_modules"
       ```

     - 这个额外的卷加载告诉 Docker 为node_modules目录使用一个标准卷，这样当npm install运行时，它不会使用主机加载

     - 当容器首次启动时，我们在entrypoint运行npm install来安装我们的依赖并填充node_modules目录

       ```bash
       entrypoint:
         - "sh"
         - "-c"
         - "npm install && ./node_modules/.bin/nodemon server.js"
       ```

## 错误 3：脆弱的配置

- 大量的复制粘贴代码，这使得代码修改非常困难

- ### 解决方案

  1. #### env 文件将环境变量从主 Docker Compose 配置中分离出来

     - 使密钥不会保存在git历史中
     - 使每个开发者拥有稍微不同的设置变得容易。例如，每个开发者可能有一个唯一的access密钥。
     - 将配置保存在一个.env文件中意味着他们不必修改提交的docker-compose.yml文件，并在这个文件更新时处理冲突。
       - 要使用 env 文件，只需增加一个.env文件，或者使用env_file字段显式设置路径

  2. #### 使用 override 文件

     - Override文件让你有一个基本配置，然后在不同文件中指定修改。
     - 如果使用 Docker Swarm，并且有一个生产环境的 YAML 文件，这将非常有用。
     - 可以在docker-compose.yml中存储自己的生产环境配置，然后在一个 override 文件中指定开发环境所需的任何更改，例如使用主机卷

  3. #### 使用extends：  用 Docker Compose v2

     - 可以使用extends关键字在多个地方导入 YAML 片段
     - Compose v3 移除了对extends关键词的支持。可以使用YAML anchors获得类似的结果

  4. #### 程序生成 Compose 文件：编写一个脚本，来基于一些高级别的规范生成 Docker Compose 文件

## 错误 4：脆弱的引导

- 假设docker-compose up，启动了一半，不得不使用docker-compose restart来启动崩溃的服务

  - 可能的原因：服务启动顺序错误有关，依赖的服务尚未启动

- ### 解决方案

  1. #### 使用depends_on

     - depends_on使你能控制启动顺序。 默认地，depends_on会等待依赖被创建，而不等待处于“healthy”状态的依赖。
     - Docker Compose v2 支持将 depends_on 与健康状态检查结合起来。
     - 这个功能在 Docker Compose v3 中被移除了。你可以使用一个类似wait-for-it.sh的脚本来手动实现类似功能

## 错误 5：资源管理不善

- 要确保 Docker 拥有它流畅运行所需的资源，而不会完全超出你的笔记本电脑负担

- 如果你觉得自己的开发工作流由于 Docker 没有以峰值容量运行而变得迟缓

- ### 解决方案

  1. #### 修改 Docker Desktop 分配

     - Docker Desktop 需要大量 RAM 和 CPU，尤其是在 Mac 和 Windows 上它是一个虚拟机时。
     - 默认的 Docker Desktop 配置往往没有分配足够的 RAM 和 CPU，因此我们一般建议调整设置到过度分配。

  2. #### 删除未使用的资源

     - 偶尔运行docker system prune，删除当前没有使用的所有卷、容器和网络。

  3. #### 在云上运行

     - Blimp，一种在云上运行 Docker Compose 文件的简单方法

## 提升 Docker Compose 上的开发者体验

- 最小化容器重新构建
- 使用主机卷
- 力求可维护的compose文件，就像代码一样。
- 使你的引导可靠
- 用心管理资源