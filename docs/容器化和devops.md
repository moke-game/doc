# 容器化和Devops

容器化技术是一种轻量级的虚拟化技术，它是一种将应用程序及其依赖打包在一起以便在不同环境中运行的技术,使得应用程序的部署和运维变得更加简单和高效。

Devops是一种软件开发方法，它强调开发团队和运维团队之间的协作和沟通，以实现快速交付高质量的软件。

容器化技术和Devops的结合，可以帮助团队更好地实现持续集成、持续交付和持续部署，从而提高软件交付的速度和质量。

## 需要了解的知识点

* [docker](https://www.docker.com/)
* [jenkins](https://www.jenkins.io/)
* [argocd](https://argo-cd.readthedocs.io/en/stable/)

## 服务容器化

1. 编辑相关服务的`Dockerfile`文件 ,放到`build/package/base`目录下

```dockerfile
## you can set APP_NAME and NETRC_CONTENT as build args
ARG APP_NAME=game
ARG NETRC_CONTENT
# Step 1: Modules caching
FROM atomhub.openatom.cn/library/golang:1-alpine3.17 as modules
RUN set -eux && sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories
MAINTAINER gstones
RUN apk add --no-cache ca-certificates git
COPY go.mod go.sum /modules/
#COPY ./cert.crt /usr/local/share/ca-certificates/my-cert.crt
RUN update-ca-certificates
#add your private repo secret as NETRC_CONTENT env,https://everything.curl.dev/usingcurl/netrc.html
#ARG NETRC_CONTENT
#RUN echo $NETRC_CONTENT > /root/.netrc
WORKDIR /modules
ENV GO111MODULE="on"
ENV GOPROXY="https://goproxy.cn,direct"
#ENV GOPRIVATE="<your private repo>"
#ENV GONOSUMDB="<your private repo>"
RUN go mod download

# Step 2: Builder
FROM atomhub.openatom.cn/library/golang:1-alpine3.17 as builder
ARG APP_NAME
COPY --from=modules /go/pkg /go/pkg
COPY . /app
WORKDIR /app
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -o /bin/app ./cmd/${APP_NAME}

# Step 3: Final
FROM alpine
RUN #sed -i -e 's/http:/https:/' /etc/apk/repositories
RUN set -eux && sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories
COPY configs /configs
RUN #apk --no-cache add tzdata
#ENV TZ=Asia/Shanghai
COPY --from=builder /bin/app /app
CMD ["/app"]
```

2. 在本地build docker镜像，以保证没有问题

 ``` shell
 # fix <appname> to service name
 # fix <your private registry> to your private registry
docker buildx build -t <your private registry> /game:latest --build-arg  NETRC_CONTENT="$NETRC_CONTENT" --build-arg APP_NAME=<appname> -f ./build/package/base/Dockerfile . --push
 ```

## CI

Test -> Build -> Push

1. 添加`Jenkinsfile`文件

```groovy
pipeline {
    agent any
    environment {
        APP_NAME = 'game'
        NETRC_CONTENT = credentials('netrc')
    }
    stages {
        stage('Test') {
            steps {
                sh 'echo "Tests"'
            }
        }
        stage('Build') {
            steps {
                script {
                    docker.build("game:${env.BUILD_NUMBER}", "--build-arg APP_NAME=${APP_NAME} --build-arg NETRC_CONTENT=${NETRC_CONTENT} -f ./build/package/base/Dockerfile .")
                }
            }
        }
        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://atomhub.openatom.cn', 'atomhub') {
                        docker.image("game:${env.BUILD_NUMBER}").push("latest")
                    }
                }
            }
        }
    }
}
```

2. 配置`Jenkins`的`Credentials`：
   在`Jenkins`的`Credentials`中添加`Secret text`类型的凭据，填写`ID`和`Secret`，`ID`需要和`Jenkinsfile`中的`credentials`
   函数中的`ID`一致

3. 配置`Jenkins`的`Pipeline`任务：
   在`Pipeline`任务中配置`Pipeline script from SCM`，选择`Git`，填写`Repository URL`和`Credentials`，选择`Jenkinsfile`路径

4. 运行`Pipeline`任务,查看`Jenkins`的`Console Output`，查看构建过程.
5. 你可以监控不同的git操作，比如`push`，`pull request`等，来触发`Jenkins`的不同的`Pipeline`任务：
    * 每次 `push`，`Jenkins`会自动触发`Pipeline`的测试任务，实现测试一切的目标
    * 每次`merge pull request `到`main`分支，`Jenkins`会自动触发`Pipeline`任务，进行构建和推送镜像
## CD

Deploy -> Smoke Test

* 测试环境的CD可以参考[10分钟搭建自动部署(CD)的单机集群环境](https://juejin.cn/post/7384303275377606667)

* 生产环境k8s集群部署建议使用[argocd](https://argo-cd.readthedocs.io/en/stable/)
  进行部署,可以参考[ArgoCD快速入门](https://argoproj.github.io/argo-cd/getting_started/)

> 注意：你需要把k8s部署的相关配置单独放到一个git仓库中，然后在argocd中配置这个git仓库的地址，argocd会自动同步这个git仓库的配置到k8s集群中
