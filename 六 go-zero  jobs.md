日常任务开放中，我们会有很多异步、批量、定时、延迟任务要处理，go-zero中有go-queue，推荐使用go-queue去处理，go-queue本身也是基于go-zero开发的，其本身是有两种模式

- dq : 依赖于beanstalkd，分布式，可存储，延迟、定时设置，关机重启可以重新执行，消息会丢失，使用非常简单，go-queue中使用了redis setnx保证了每个消息只被消费一次，使用场景主要是用来做日常任务使用
- kq：依赖于kafka，这个就不多介绍啦，大名鼎鼎的kafka，使用场景主要是做日志用

我们主要说一下dq，kq使用也一样的，只是依赖底层不同，如果没使用过beanstalkd，没接触过beanstalkd的可以先google一下，使用起来还是挺容易的。

我在jobs下使用goctl新建了一个message-job.api服务

```go
info(
	title: //消息任务
	desc: // 消息任务
	author: "Mikael"
	email: "13247629622@163.com"
)

type BatchSendMessageReq {}

type BatchSendMessageResp {}

service message-job-api {
	@handler batchSendMessageHandler // 批量发送短信
	post batchSendMessage(BatchSendMessageReq) returns(BatchSendMessageResp)
}
```



因为不需要使用路由，所以handler下的routes.go被我删除了，在handler下新建了一个jobRun.go，内容如下：

```go
package handler

import (
	"fishtwo/lib/xgo"
	"fishtwo/app/jobs/message/internal/svc"
)


/**
* @Description 启动job
* @Author Mikael
* @Date 2021/1/18 12:05
* @Version 1.0
**/

func JobRun(serverCtx *svc.ServiceContext)  {

	xgo.Go(func() {
		batchSendMessageHandler(serverCtx)
    //...many job
	})
}
```

其实xgo.Go就是 go batchSendMessageHandler(serverCtx) ，封装了一下go携程，防止野生goroutine panic



然后修改一下启动文件message-job.go

```go
package main

import (
   "flag"
   "fmt"

   "fishtwo/app/jobs/message/internal/config"
   "fishtwo/app/jobs/message/internal/handler"
   "fishtwo/app/jobs/message/internal/svc"

   "github.com/tal-tech/go-zero/core/conf"
   "github.com/tal-tech/go-zero/rest"
)

var configFile = flag.String("f", "etc/message-job-api.yaml", "the config file")

func main() {
   flag.Parse()

   var c config.Config
   conf.MustLoad(*configFile, &c)

   ctx := svc.NewServiceContext(c)
   server := rest.MustNewServer(c.RestConf)
   defer server.Stop()

   handler.JobRun(ctx)

   fmt.Printf("Starting server at %s:%d...\n", c.Host, c.Port)
   server.Start()
}
```

主要是handler.RegisterHandlers(server, ctx) 修改为handler.JobRun(ctx)



接下来，我们就可以引入dq了，首先在etc/xxx.yaml下添加dqConf

```yaml
.....

DqConf:
  Beanstalks:
    - Endpoint: 127.0.0.1:7771
      Tube: tube1
    - Endpoint: 127.0.0.1:7772
      Tube: tube2
  Redis:
    Host: 127.0.0.1:6379
    Type: node

```

我这里本地用不同端口，模拟开了2个节点，7771、7772

在internal/config/config.go添加配置解析对象

```go
type Config struct {
	....
	DqConf dq.DqConf
}

```



修改handler/batchsendmessagehandler.go

```go
package handler

import (
	"context"
	"fishtwo/app/jobs/message/internal/logic"
	"fishtwo/app/jobs/message/internal/svc"
	"github.com/tal-tech/go-zero/core/logx"
)

func batchSendMessageHandler(ctx *svc.ServiceContext){

	rootCxt:= context.Background()
	l := logic.NewBatchSendMessageLogic(context.Background(), ctx)
	err := l.BatchSendMessage()
	if err != nil{
		logx.WithContext(rootCxt).Error("【JOB-ERR】 : %+v ",err)
	}
}

```



修改logic下batchsendmessagelogic.go，写我们的consumer消费逻辑

 ```go
package logic

import (
	"context"
	"fishtwo/app/jobs/message/internal/svc"
	"fmt"
	"github.com/tal-tech/go-zero/core/logx"
)

type BatchSendMessageLogic struct {
	logx.Logger
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

func NewBatchSendMessageLogic(ctx context.Context, svcCtx *svc.ServiceContext) BatchSendMessageLogic {
	return BatchSendMessageLogic{
		Logger: logx.WithContext(ctx),
		ctx:    ctx,
		svcCtx: svcCtx,
	}
}


func (l *BatchSendMessageLogic) BatchSendMessage() error {

	fmt.Printf("job BatchSendMessage start \n")

	l.svcCtx.Consumer.Consume(func(body []byte) {
		fmt.Printf("job BatchSendMessage %s \n" + string(body))
	})

	fmt.Printf("job BatchSendMessage finish \n")
	return nil
}

 ```



这样就大功告成了，启动message-job.go就ok课

```shell
go run message-job.go
```



之后我们就可以在业务代码中向dq添加任务，它就可以自动消费了



 producer.Delay 向dq中投递5个延迟任务：

```go
	producer := dq.NewProducer([]dq.Beanstalk{
		{
			Endpoint: "localhost:7771",
			Tube:     "tube1",
		},
		{
			Endpoint: "localhost:7772",
			Tube:     "tube2",
		},
	})

	for i := 1000; i < 1005; i++ {
		_, err := producer.Delay([]byte(strconv.Itoa(i)), time.Second * 1)
		if err != nil {
			fmt.Println(err)
		}
	}
```

 producer.At可以指定某个时间执行，非常好用，感兴趣的朋友自己可以研究下





























