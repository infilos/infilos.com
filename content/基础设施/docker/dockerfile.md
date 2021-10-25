---
type: docs
title: "Dockerfile"
linkTitle: "Dockerfile"
weight: 1
---

## Dockerfile

Docker可以通过`Dockerfile`自动构建镜像，`Dockerfile`是一个包含多个指令的文档。如下

```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

## FROM

`FROM`命令用于初始化一个新的构建阶段，并为后续指令设置基础镜像：

```dockerfile
FROM [--platform=<platform>] <image> [AS <name>]
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
```

- `FROM`指令用于指定基础镜像，`ARG`是唯一可以位于`FROM`指令之前的指令，一般用于声明基础镜像的版本。
- 单个`Dockerfile`可以多次出现`FROM`，以使用之前的构建阶段作为另一个构建阶段的依赖项。
- `--platform`选项可用在`FROM`多平台镜像的情况下指定平台。例如，linux/amd64、lunux/arm64、windows/amd64。
- `AS name`表示为构建阶段命令，在后续`FROM`和`COPY --from=name`说明中可以使用这个名词，引用此阶段构建的映像。
- `tag`或`digest`值是可选的。如果您省略其中任何一个，构建器默认使用`latest`标签。如果找不到指定`tag`，构建起将返回错误。

```dockerfile
ARG CODE_VERSION=latest
FROM base:${CODE_VERSION}
CMD /code/run-app
FROM extras:${CODE_VERSION}
CMD /code/run-extras
```

## RUN

`RUN`指令将在当前镜像之上的新层中执行命令，并且提交结果。在`docker build`时运行。

```dockerfile
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'
```

RUN有两种形式：

- RUN <command>：shell形式，命令在shell中运行，默认在Linux上使用`/bin/sh -c`，在Windows上使用`cmd /S /C`。
- RUN ["程序名","param1","param1"]：exec形式，不会触发shell，所以$HOME这样的环境变量无法使用，但它可以在没有bash的镜像中执行。

说明：

- 可以使用反斜杠（\）将单个`RUN`命令延续到下一行。
- `RUN`在下一次构建期间，指令缓存不会自动失效。可以使用`--no-cache`选项使指令缓存失效。如RUN apt-get update之类的构建缓存将在下一次构建期间被重用，此时构建中可能安装过时版本的工具，但我们可以使用--no-cache标志来使RUN命令的缓存失效，如docker build --no-cache。
- Dockerfile的指令每执行一次就会给镜像新建一层只读层。过多无意义的层会造成镜像膨胀过大，可以使用&&符号连接多个命令，这样执行RUN指令后之后创建一层镜像。

有些命令会使用管道（|），如：

```dockerfile
RUN wget -O - https://some.site | wc -l > /number
```

Docker使用`/bin/sh -c`解释器，解释器只计算所有一个命令的退出状态码以确定命令是否执行成功，如上例，只要`wc -l`执行成功，即使`wget`命令失败，这个构建步骤也会生成一层新镜像。如果希望命令由于管道中任意阶段命令的错误而失败，需要预先设置`set-o pipefail&&`，如下：

```
 RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```

> 注意，基于debian的镜像要使用exec形式支持`-o pipefail`:
>
> ```dockerfile
> RUN ["/bin/bash", "-c", "set -o pipefail && wget -O - https://some.site | wc -l > /number"]
> ```

## CMD

Dockerfile使用`RUN`指令完成`docker build`所有的环境安装与配置，通过`CMD`指令来指示`docker run`命令运行镜像时要执行的命令。Dockerfile只允许使用一次`CMD`命令。使用多个`CMD`会抵消之前所有的命令，只有最后一个命令生效。一般来说，这是整个Dockerfile脚本的最后一个命令。

```dockerfile
FROM ubuntu
CMD ["/usr/bin/wc","--help"]
```

CMD有三种形式：

- CMD ["exec","param1","param2"]：使用exec执行，推荐方式。
- CMD command param1 param2：在/bin/sh中执行，可以提供交互操作。
- CMD ["param1","param2"]：提供给ENTRYPOINT的默认参数（极少这样使用）。

## EXPOSE

`EXPOSE`指令通知容器在运行时监听某个端口，可以指定TCP或UDP，如果不指定协议，默认为TCP端口。但是为了安全，`docker run`命令如果没有带上相应的端口映射参数，Docker并不会将端口映射出去。

```dockerfile
EXPOSE 80/tcp
EXPOSE 80/udp
```

指定映射端口方式：

docker run -P：将所有端口发布到主机接口，每个公开端口绑定到主机上的随机端口，端口范围在`/proc/sys/net/ipv4/ip_local_port_range`定义的临时端口范围内。

docker run -p ：显式映射单个端口或端口范围。

## LABEL

`LABEL`指令为镜像添加标签，当前镜像会继承父镜像的标签，如果与父标签重复，会覆盖之前的标签。

```dockerfile
LABEL multi.label1="value1" \
  multi.label2="value2" \
  other="value3"
# 或
LABEL multi.label1="value1" multi.label2="value2" other="value3"
```

可以使用如下方式查看镜像标签：

```bash
docker image inspect --format='' myimage
```

查看结果实例：

```json
{
 "com.example.vendor": "ACME Incorporated",
 "com.example.label-with-value": "foo",
 "version": "1.0",
 "description": "This text illustrates that label-values can span multiple lines.",
 "multi.label1": "value1",
 "multi.label2": "value2",
 "other": "value3"
}
```

## ENV

`ENV`命令用来在执行`docker run`命令运行镜像时指定自动设置的环境变量。这个环境变量可以在后续任何RUN命令中使用，并在容器运行时保持。一般用于软件更便捷的运行，如：

```dockerfile
ENV PATH=/usr/local/nginx/bin:$PATH
CMD ["nginx"]
```

设置的环境变量将持续存在，可以使用`docker inspect`来查看。这些环境变量可以通过`docker run --env <key>=<value>`命令的参数来修改。

## ARG

ARG命令定义用户只在构建时使用的变量，效果和`docker build --build-arg <varname>=<value>`一样，这个参数只会在构建时存在，不会保留在镜像中。

```dockerfile
ARG <name>[=<default value>]
```

ARG与ENV类似，不同的是ENV会在镜像构建结束后一直保存在容器中，而ARG会在镜像构建结束狗消失。一般运用在希望整个构建过程是无交互的，那么可以使用ARG命令（仅限Debian发行版）。

```dockerfile
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y ...
```

Docker 有一组预定义的 ARG 变量，您可以在 Dockerfile 中没有相应指令的情况下使用这些变量。

- HTTP_PROXY
- http_proxy
- HTTPS_PROXY
- https_proxy
- FTP_PROXY
- ftp_proxy
- NO_PROXY
- no_proxy

要使用这些，请使用 --build-arg 标志在命令行上传递它们，例如：

```bash
docker build --build-arg HTTPS_PROXY=https://my-proxy.example.com .
```

## ADD

`ADD`指令用于复制新文件、目录或远程文件 URL到容器<dest>路径中。可以指定多个资源，但如果它们是文件或目录，则它们的路径被解释为相对于构建上下文的源，也就是 WORKDIR。

`ADD`指令有两种形式：

```dockerfile
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

每个都 <src> 可能包含通配符，匹配将使用 Go 的 filepath.Match 规则。<dest> 是一个绝对路径，或相对 WORKDIR 的相对路径。例如：

添加所有以“hom”开头的文件：

```dockerfile
ADD hom* /mydir/
```

在下面的示例中，? 被替换为任何单个字符，例如“home.txt”。

```dockerfile
ADD hom?.txt /mydir/
```

所有新创建的文件和目录的UID和GID都为0，除非使用`--chown`指定UID/GID以及权限。`-chown`特性仅在用于构建Linux容器。

```dockerfile
ADD --chown=55:mygroup files* /somedir/
ADD --chown=bin files* /somedir/
ADD --chown=1 files* /somedir/
ADD --chown=10:11 files* /somedir/
```

如果<src>是一个可识别压缩格式（identity、gzip、bzip2或xz）的本地tar存档，那么它将被解包为一个目录。远程URL中的资源不会解压缩。当一个目录被复制或解包时，它的行为与`tar -x`相同。

## COPY

COPY 指令和 ADD 指令的唯一区别在于：是否支持从远程URL获取资源。COPY 指令只能从执行 docker build 所在的主机上读取资源并复制到镜像中。而 ADD 指令还支持通过 URL 从远程服务器读取资源并复制到镜像中。

相同复制命令下，使用ADD构建的镜像比COPY命令构建的体积大，所以如果只是复制文件使用COPY命令。ADD 指令更擅长读取本地tar文件并解压缩。

## ENTRYPOINT

`ENTRYPOINT`和`CMD`一样，都是在指定容器启动程序以及参数，不会它不会被`docker run`的命令行指令所覆盖。如果要覆盖的话需要通过`docker run --entrypoint`来指定。

ENTRYPOINT有两种形式：

```dockerfile
ENTRYPOINT ["exec","param1","param1"]
ENTRYPOINT command param1 param2
```

指定了ENTRYPOINT后，CMD的内容作为参数传递给ENTRYPOINT指令，实际执行时将变为：

```
<ENTRYPOINT><CMD>
```

## VOLUME

创建一个具有指定名称的挂载数据卷。

```dockerfile
VOLUME ["/var/log/"]
VOLUME /var/log
```

VOLUME的主要作用是：

- 避免重要的数据因容器重启而丢失。
- 避免容器不断变大。
- 保留配置文件。

## ONBUILD

`ONBUILD`指令作为触发指令添加到镜像中，只有在该镜像作为基础镜像时执行。触发器将在下游构建的Dockerfile中的`FROM`指令之后执行。如果任何触发器失败，`FROM`指令将中止，从而导致生成失败。如果所有触发器都成功，`FROM`指令将完成，构建将照常继续。

```dockerfile
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
```

> 注意，ONBUILD指令不能触发FORM和MAINTAINER指令。

## STOPSIGNAL

设置容器退出时唤起的系统调用信号。该信号可以是与内核系统调用表中的位置匹配的有效无符号数字，例如9，或格式为SIGNAME的信号名称，如SIGKILL。

```dockerfile
STOPSIGNAL signal
```

默认的stop-signal是SIGTERM，在`docker stop`的时候会给容器内PID为1的进程发送这个信号，通过`--stop-signal`可以设置需要的signal，主要用于让容器内的程序在接收到signal之后可以先处理些未完成的事务，实现优雅结束进程后退出容器。如果不做任何处理，容器将在一段时间后强制退出，可能会造成业务强制中断，默认时间是10s。

## HEALTHCHECK

`HEALTHCHECK`指令告诉容器如何检查它是否保持运行。当容器具有指定的`HEALTHCHECK`时，除了其正常状态外，还具有健康状态。容器的状态最初是`starting`，只要通过健康检查，容器的状态就变成`healthy`（无论之前处于什么状态）。如果经过一定数量的失败检查，容器的状态会变成`unhealthy`。

该`HEALTHCHECK`指令有两种形式：

- HEALTHCHECK [OPTIONS] CMD command：通过容器内运行命令来检查容器健康状况。后面命令的退出状态会影响容器的健康状态，如：

- - 0: success - 容器是健康的，并且准备使用
  - 1: unhealthy - 容器没有正确工作
  - 2: reserved - 没有使用退出状态

- HEALTHCHECK NONE：禁用从基础镜像继承的任何健康检查。

`HEALTHCHECK`选项（应处于`CMD`之前）：

- `--interval=DURATION`：检查间隔，default: `30s`。
- `--timeout=DURATION`：检查超时时间，超出此范围认为检查失败，default: `30s`。
- `--start-period=DURATION`：容器初始化阶段的时间，此阶段健康检查失败不计入最大重试次数，如果检查成功则认为容器已启动，default: `0s`。
- `--retries=N`：健康检查连续失败次数，default: `3`。

举例：

```dockerfile
HEALTHCHECK --interval=5m --timeout=3s \
 CMD curl -f http://localhost/ || exit 1
```

## SHELL

`SHELL`指令用于设置默认shell。Linux上默认shell是`["/bin/sh","-c"]`，Windows上是`["cmd","/S","/C"]`。

```dockerfile
SHELL ["exec","param1"]
```

`SHELL`指令在Windows上特别有用，因为Windows有两种截然不同的本机SHELL：CMD和powershell，以及备用的sh。该SHELL指令可以出现多次。每条SHELL指令都会覆盖所有先前的SHELL指令，并影响后续指令。

```dockerfile
FROM
Learn more about the "FROM" Dockerfile command.
 microsoft/windowsservercore

# Executed as cmd /S /C echo default
RUN echo default

# Executed as cmd /S /C powershell -command Write-Host default
RUN powershell -command Write-Host default

# Executed as powershell -command Write-Host hello
SHELL ["powershell", "-command"]
RUN Write-Host hello

# Executed as cmd /S /C echo hello
SHELL ["cmd", "/S", "/C"]
RUN echo hello
```

## WORKDIR

`WORKDIR`指令为Dockerfile中接下来的`RUN`、`CMD`、`ENTRYPOINT`、`ADD`、`COPY`指令设置工作目录。如果`WORKDIR`不存在，及时它没有在后续Dockerfile指令中使用，它也会被创建。

Dockerfile中可以多次使用`WORKDIR`，如果提供了相对路径，它将相对于前一条`WORKDIR`指令的路径。

```dockerfile
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```

最终 pwd 命令的输出是 /a/b/c。

该 WORKDIR 指令可以解析先前使用 ENV，例如：

```dockerfile
ENV DIRPATH=/path
WORKDIR $DIRPATH/$DIRNAME
RUN pwd
```

最终 pwd 命令的输出是 /path/$DIRNAME。

> 官方推荐WORKDIR始终使用绝对路径。此外，尽量避免使用`RUN cd ..&& dosomething`，大量的类似指令会降低可读性，令Dockerfile难以维护。	

## USER

`RUN`指令设置用户名或（UID）和可选用户组（或GID），用于运行`Dockerfile`中接下来的`RUN`、`CMD`、`ENTRYPOINT`指令。

```dockerfile
USER <user>[:<group>]
USER <UID>[:<GID>]
```

> 注意，在Linux上，当用户没有主组时，镜像（或指令）将与根组一起运行。在Windows上，如果用户不是内置帐户，则必须首先创建该用户。也可以先通通过`net user`创建用户，再指定用户。
>
> ```dockerfile
> FROM microsoft/windowsservercore
> # Create Windows user in the container
> RUN net user /add patrick
> # Set it for subsequent commands
> USER patrick
> ```

## MAINTAINER

MAINTAINER指令设置生成镜像的作者。如：

```dockerfile
MAINTAINER <name>
```

