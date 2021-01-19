在介绍go-zero实际使用前，先说一下整体架构，更方便理解 

![go-zero-k8s架构](./images/二/go-zero-arch.png)

### CI/CD

Step1：本地deveploer开发好代码之后提交到gitlab（这里分支就不详细说明了）

Step2：jenkins，使用pipline方式部署

- 从gitlab拉取代码
- docker build ，基于最新gitlab上的code构建镜像
- docker push，将构建好的镜像推送到镜像仓库（当然一般都是有自己私有镜像仓库比如harbor，用阿里云的也可以）
- kubectl apply -f xxx.yaml  ：使用kubectl 部署到k8s中 （阿里云k8s容器服务那么好用，不用岂不可惜？）

kubectl部署之后，k8s就会根据你的service中的yaml定义的镜像来你的镜像仓库拉取刚才你打包的最新镜像，so～～上线成功啦！

嗯，有的同学说，阿里云k8s好用是好用，可是我不会写or不想写Dockerfile，不会写k8s的yaml or 不想写，没关系，goctl说放开它，让我来

生成 Dockerfile 

```shell
$ goctl docker -go user.go 
```

生成k8s yaml

```shell
$ goctl kube deploy -name user-api -namespace blog -image user:v1 -o user.yaml -port 2233
```

所以，就是这么简单





### 访问流程

app/web/pc 透过防火墙，首先访问到阿里云的负载均衡SLB，同时SLB可以将你的后端服务器ip隐藏起来，同时可以预防DDOS攻击，虽然有额度的，但是好过没有～～，然后SLB访问到前面的nginx，nginx作为代理使用，k8s中的service通过 nodeport方式暴露出来在nignx中代理到该service，同时在nginx中上报日志到kafka，然后api可以在etcd中拿到多个rpc节点，调用多个后端rpc服务，rpc负责跟db交互、或者调用其他rpc获取数据（当然api、rpc之间是通过etcd动态发现的）返回给api，api就是聚合数据，然后层层返回到客户端。



整体架构都是高可用高可用～















