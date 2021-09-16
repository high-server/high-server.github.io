---
title: "手把手教你用TARS"
date: 2021-09-16T17:14:16+08:00
draft: false
categories:
  - "实战"

#Comment
gitment: true
---

在中国，有一个简单的方法可以用来判断一个互联网公司够不够大，那就是看其是否开源过 rpc 框架！比如阿里巴巴的 [dubbo](https://github.com/apache/dubbo)，或者腾讯的 [tars](https://github.com/TarsCloud/Tars)，小公司往往会对这些大公司的产品趋之若鹜，不过一个可悲的现实是大公司自己往往并不用他们开源的版本，这就好比皇帝总是把自己看不上眼的女人赏赐给臣民，不过能得到皇帝的赏赐总是好事，下面让我手把手教你用 tars，更具体的说是 [tarsgo](https://github.com/TarsCloud/TarsGo)，也就是 tars 的 golang 实现。

实际动手前，最好熟读[官方文档](https://github.com/TarsCloud/TarsDocs/blob/master/SUMMARY.md)，特别是[基础概念](https://github.com/TarsCloud/TarsDocs/blob/master/base/tars-concept.md)和[基础通讯协议](https://github.com/TarsCloud/TarsDocs/blob/master/base/tars-protocol.md)部分，假设你已经了解了这些内容，那么不妨让我们虚拟一个例子：给商城里的用户加积分！然后我们要构建一个 Shop（App），其中有一个 User（Server），其中有一个 Credit（Servant），可以简单的把 App、Server、Servant 这些概念理解成命名空间的几个层级，下面让我们用 tarsgo 内置的 create_tars_server_gomod.sh 脚本来生成项目！

```shell
shell> export GOPATH=$(go env GOPATH)
shell> go env -w GO111MODULE=auto
shell> go get -u github.com/TarsCloud/TarsGo/tars
shell> $GOPATH/src/github.com/TarsCloud/TarsGo/tars/tools/create_tars_server_gomod.sh Shop User Credit foo
```

因为 tarsgo 依赖 GOPATH 和 GO111MODULE，所以务必按照上面的步骤来操作，完成后会生成一个名为 User 的目录，其中的大概内容如下：

```shell
shell> tree ./User
./User
├── Credit.tars
├── Credit_imp.go
├── client
│   └── client.go
├── config.conf
├── debugtool
│   └── dumpstack.go
├── go.mod
├── main.go
├── makefile
└── start.sh
```

编辑 Credit.tars 文件，定义好我们加积分的 Add 方法：

```
module Shop
{
    interface Credit
    {
        int Add(int a,int b,out int c); // Some example function
    };
};
```

接下来就可以通过 tars 文件来生成 golang 文件了（其中调用了 tars2go 工具）：

```shell
shell> cd User
shell> go mod tidy
shell> make TARSBUILD
```

文件生成好之后，编辑 Credit_imp.go 文件，加入我们的业务逻辑：

```golang
func (imp *CreditImp) Add(ctx context.Context, a int32, b int32, c *int32) (int32, error) {
	log.Println("call add")
	return 100, nil
}
```

编译服务端和客户端代码，运行就能看到效果了：

```shell
shell> go build -o user_server
shell> go build -o user_client ./client/client.go
shell> ./user_server -config config.conf
shell> ./user_client
```

BTW：缺省生成的客户端代码 client.go 导入的包路径不正确，需要手动调整一下。

总结一下，tarsgo 的开发过程比较简单，基本上就是：编写 tars 文件；用 tars2go 生成代码；实现业务逻辑。当然了，实际部署的时候会有一个管理平台，服务治理等复杂的问题都隐藏在平台里，这些细节就不是本文所考虑的了，再见。