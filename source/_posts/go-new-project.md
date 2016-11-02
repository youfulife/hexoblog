title: go 小工程管理
date: 2016-09-07 16:51:01
tags: go
categories: 编程
---
由于go的包管理依赖于GOPATH环境变量，目前对GO的工程组织并没有摸的特别透，而且只做过一个简单的小项目。
简单说一下go项目使用gitlab管理时的一些布局和部署方式，尽量做到clone下来后一键可以编译运行。

## 目录结构如下：

```
fair
├── README.md
├── config.toml
├── install.sh
├── logs
└── src
    ├── fair
    │   ├── config.go
    │   ├── decode.go
    │   ├── dump_es.go
    │   ├── input_kafka.go
    │   └── main.go
    ├── grok
    ├── ipsearch
    └── useragent
```

其中项目的名字是fair，所有的go源文件都放在src下面，其中src/fair作为main模块，其余几个文件夹作为其它自己维护的模块。

在fair的目录下有一个install.sh这个里面放了当前项目从git clone下来后到直接运行的所有命令。

```
export GOPATH=$PWD

gofmt -w src

# golang.org/x/net has been blocked by GFW
if [ ! -d $GOPATH/src/golang.org/x ]; then
mkdir -p $GOPATH/src/golang.org/x
git clone https://github.com/golang/net $GOPATH/src/golang.org/x/net
fi

go get -v fair

# run test
# go get -v github.com/stretchr/testify
# go test fair
go test grok
go test ipsearch
go test useragent

go install -v fair

# builds for multiple platforms
# go get github.com/mitchellh/gox
# ./bin/gox -os="linux" -arch="amd64" -output="bin/fair_linux_amd64" fair

export GOPATH=$OLDGOPATH

echo 'finished'

```

每次修改完源代码后直接运行这个脚本就可以了，运行完成后会在当前项目的根目录下生成bin和pkg文件，第三方的依赖包也会放到src下面，看起来很统一，而且所有的依赖包都在当前项目的目录下。一定要在项目的根目录下运行install.sh，否则GOPATH就不会设置到项目的根目录下了。

install.sh 脚本中包含了设置GOPATH环境变量，处理GFW屏蔽的问题，下载第三方依赖，运行单元测试，编译安装二进制程序，交叉编译目标平台的二进制程序，还原GOPATH环境变量。

编译安装完成后整个项目的目录结构如下：

```
fair
├── bin
├── logs
├── pkg
└── src
    ├── fair
    ├── github.com
    ├── golang.org
    ├── gopkg.in
    ├── grok
    ├── ipsearch
    └── useragent

```

自己的代码还是自己的代码，第三方的代码还是第三方的代码，互不干扰，但是层次结构都保持一致。这样只要在项目目录下运行./bin/fair就可以了。

当初的目标就是每个人用四个命令就可以完全运行起来程序。

```
git clone git@192.168.10.43:BigData/fair.git
cd fair
./install.sh
./bin/fair

```

如果是服务端程序，那么可以使用supervisor管理起来，配置可以如下：

```
[program:fair]
command=/root/fair/bin/fair
startsecs=3
autostart=true
autorestart=true
startretries=3
stdout_logfile=/root/fair/logs/fair.log
stderr_logfile=/root/fair/logs/fair.log
directory=/root/fair

```

目前团队的两个项目都是基于这种目录结构的管理方式，感觉还是很优雅的。
