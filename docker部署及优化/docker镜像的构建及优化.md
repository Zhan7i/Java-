## 源程序

```go
/* hello.c */
int main () {
  puts("Hello, world!");
  return 0;
}
```

## Dockerfile 构建镜像

```bash
FROM gcc     # 定制的镜像都是基于 FROM 的镜像
COPY hello.c .  #  复制指令，从上下文目录中复制文件或者目录到容器里指定路径。
RUN gcc -o hello hello.c  # <命令行命令> 等同于，在终端操作的 shell 命令。
CMD ["./hello"]   运行容器时执行的shell命令
```

- ### 一些dockerFile参数 

> Dockerfile  部分相关指令说明
> FROM：构建镜像基于哪个镜像
> MAINTAINER: 	镜像维护者姓名或邮箱地址
> RUN：构建镜像时运行的指令
> CMD：运行容器时执行的shell环境
> VOLUME：指定容器挂载点到宿主机自动生成的目录或其他容器
> USER：	为RUN、CMD、和 ENTRYPOINT 执行命令指定运行用户
> WORKDIR：为 RUN、CMD、ENTRYPOINT、COPY 和 ADD 设置工作目录，就是切换目录
> HEALTHCHECH：健康检查
> ARG：构建时指定的一些参数
> EXPOSE：声明容器的服务端口（仅仅是声明）
> ENV：设置容器环境变量
> ADD：拷贝文件或目录到容器中，如果是URL或压缩包便会自动下载或自动解压
> COPY：拷贝文件或目录到容器中，跟ADD类似，但不具备自动下载或解压的功能
> ENTRYPOINT：运行容器时执行的shell命令

## 如何减小镜像的大小：多阶段构建：注意docker 版本需要1.17及以上

- 多阶段构建可以由多个 FROM 指令识别，每一个 FROM 语句表示一个新的构建阶段，阶段名称可以用 AS 参数指定

  ```bash
  FROM gcc AS mybuildstage
  COPY hello.c .
  RUN gcc -o hello hello.c
  FROM ubuntu
  COPY --from=mybuildstage hello .
  CMD ["./hello"]
  ```

  - 使用基础镜像 gcc 来编译程序 hello.c，然后启动一个新的构建阶段，它以 ubuntu 作为基础镜像，将可执行文件 hello 从上一阶段拷贝到最终的镜像中

  - 在声明构建阶段时，可以不必使用关键词 AS，最终阶段拷贝文件时可以直接使用序号表示之前的构建阶段（从零开始）

    - 下面两行是等效的

      ```bash
      COPY --from=mybuildstage hello .
      COPY --from=0 hello .
      ```

- 在构建的第一阶段使用经典的基础镜像，这里经典的镜像指的是 CentOS，Debian，Fedora 和 Ubuntu 之类的镜像

- ### COPY --from 使用绝对路径

  - 从上一个构建阶段拷贝文件时，使用的路径是相对于上一阶段的**根目录**的

  - 最好的方法是在第一阶段指定 WORKDIR，在第二阶段使用绝对路径拷贝文件，这样即使基础镜像修改了 WORKDIR，也不会影响到镜像的构建

    ```bash
    FROM golang
    WORKDIR /src
    COPY hello.go .
    RUN go build hello.go
    FROM ubuntu
    COPY --from=0 /src/hello .
    CMD ["./hello"]
    ```

- ### FROM scratch

  - 将多阶段构建的第二阶段的基础镜像改为 scratch。scratch 是一个虚拟镜像，不能被 pull，也不能运行

    - 因为它表示空、nothing！这就意味着新镜像的构建是从零开始，不存在其他的镜像层

  - ```bash
    FROM golang
    COPY hello.go .
    RUN go build hello.go
    FROM scratch
    COPY --from=0 /go/hello .
    CMD ["./hello"]
    ```

  - #### 使用scratch作为基础镜像的缺点

    - 缺少shell： CMD/RUN 语句中不能使用字符串，镜像中并不包含 /bin/sh，所以无法运行程序

      - 如何解决：使用 JSON 语法取代字符串语法：将 CMD ./hello 替换为  CMD ["./hello"]

    - 缺少调试工具：无法使用 docker exec 进入容器，也无法查看网络堆栈信息

      - 想查看容器中的文件，可以使用 docker cp；如果想查看或调试网络堆栈，可以使用 docker run --net container:，或者使用 nsenter；

    - 可以选择 busybox 或 alpine 镜像来替代 scratch

    - 缺少libc：C 语言版本和复杂的go程序无法执行

      - 解决方法

        - 使用静态库：gcc -o hello hello.c -static   #这会使得可执行文件中包含了其运行所需要的库文件

        - 拷贝库文件到镜像中：为了找出程序运行需要哪些库文件，可以使用 ldd 工具：

          ```bash
          $ ldd hello
              linux-vdso.so.1 (0x00007ffdf8acb000)
              libc.so.6 => /usr/lib/libc.so.6 (0x00007ff897ef6000)
              /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007ff8980f7000)
           #从输出结果可知，该程序只需要 libc.so.6 这一个库文件。linux-vdso.so.1 与一种叫做 VDSO 的机制有关，用来加速某些系统调用，可有可无。ld-linux-x86-64.so.2 表示动态链接器本身，包含了所有依赖的库文件的信息
           #该方法非常难以维护，后期需要不断地更改，而且还有很多未知的隐患。
          ```

        - 使用 busybox:glibc 作为基础镜像：选择一个合适的镜像来运行使用动态链接的程序，busybox:glibc 是最好的选择

## 总结

- 不同构建方法构建的镜像大小：
  - 原始的构建方法：1.14 GB
  - 使用 ubuntu 镜像的多阶段构建：64.2 MB
  - 使用 alpine 镜像和静态 glibc：6.5 MB
  - 使用 alpine 镜像和动态库：5.6 MB
  - 使用 scratch 镜像和静态 glibc：940 kB
  - 使用 scratch 镜像和静态 musl libc：94 kB
  
- 不建议使用 sratch 作为基础镜像，因为调试起来非常麻烦

- ### 当已经构建过 存在缓存时，可以使用命令

  - docker build --target runner  -t core-alpine 利用缓存构建镜像