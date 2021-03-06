# 在 Docker 容器中运行多个进程

##### 作者：[Alexander Beletsky](https://github.com/alexanderbeletsky)

##### 译者：[巨震](https://github.com/crystaldust)

我最喜欢 Docker 的地方就是它那全新的软件部署和发布方式。我经常会看到令人兴奋的软件，兴冲冲的准备研究一下，然后被其安装说明在第一时间扼杀掉我的兴趣。稍微大一点的软件需要很多附属条件：运行时间、库、数据库。


在Docker中，这些安装步骤被精简成这样子：

```
    $ docker pull vendor/package
    $ docker run vendor/package
```

基本上就是这么几行命令了，不需要你在服务器上装什么 Java 运行时环境，非常适合 TCP/HTTP 的程序。


尽管被自己的 [Seismo](https://github.com/seismolabs/seismo) 项目搞的焦头烂额后，我还是希望能够按原计划把它实现。因为这个项目的条件很少，只有 MongoDB 和 NodeJS ，任何人都可以很轻松的上手，虽然他们不经常安装设置。令人开心的是，Github 现在对 Docker 提供了很好地支持，它会检测你的项目，如果发现其中有 `Dockerfile`，那么当你 push 代码的时候，Docker 镜像会自动重建然后 Github 会把新的镜像 push 到 Docker 的 [公共镜像列表](https://index.docker.io/) 中。

我创建了一个 `Dockerfile`，可以构建一个运行着Seismo的镜像。

```
    FROM    ubuntu:latest
    
    # Git
    RUN apt-get install -y git
    
    # MongoDB
    RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
    RUN echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | tee /etc/apt/sources.list.d/10gen.list
    RUN dpkg-divert --local --rename --add /sbin/initctl
    RUN ln -s /bin/true /sbin/initctl
    RUN apt-get update
    RUN apt-get install mongodb-10gen
    RUN mkdir -p /data/db
    
    # NodeJS
    RUN apt-get update --fix-missing && apt-get upgrade -y
    RUN apt-get install -y wget curl build-essential patch git-core openssl libssl-dev unzip ca-certificates
    RUN curl http://nodejs.org/dist/v0.10.22/node-v0.10.22-linux-x64.tar.gz | tar xzvf - --strip-components=1 -C "/usr"
    RUN apt-get clean && rm -rf /var/cache/apt/archives/* /var/lib/apt/lists/*
    
    # Seismo
    RUN git clone https://github.com/seismolabs/seismo.git /seismo
    RUN cd /seismo; npm install
    ENV PORT 8080
    EXPOSE 8080
    
    WORKDIR /seismo
    ENTRYPOINT ["./bin/run.sh"]
```

这个镜像基于最新的服务器版 Ubuntu，安装了 Git，MongoDB 和 NodeJS 的运行环境，并且把 Seismo 的库克隆到镜像内部。

但是，在容器内部启动多个进程的时候，我遇到了一些问题。因为我需要 MongoDB 作为存储，还需要 NodeJS 做 API 服务器，需要在容器中同时运行这两个程序。如果shell脚本只启动了一个，比如，只启动了 `mongod`，那么`node app.js`就不会执行。

我有点儿小慌张，以为容器内部无法同时运行多个进程。

但我还是找到了解决方案。我重新写了脚本，把 mongod 作为后台进程开起来，然后再启动 nodejs 。

```
    #!/bin/bash
    mongod & node ./source/server.js
```

问题完美解决！

---
##### 这篇文章由 [Alexander Beletsky](https://github.com/alexanderbeletsky) 发表，点击 [此处](http://beletsky.net/2013/12/run-several-processes-in-docker-container.html) 可查阅原文。

##### The article was contributed by [Alexander Beletsky](https://github.com/alexanderbeletsky) , click [here](http://beletsky.net/2013/12/run-several-processes-in-docker-container.html) to read the original publication.
