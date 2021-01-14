![1610622907172](/Users/seven/Desktop/go-zero文章/images/五/1610622907172.jpg)

model：rpc服务中，官方文档推荐是将model放在services目录下，与每个rpc服务一层，但是个人感觉每个model对应一张表，一张表只能由一个服务去控制，哪个服务控制这张表就哪个服务拥有控制这个model权利，其他服务想访问就要通过grpc，这是个人的想法，所以我把每个服务自己管控的model放在了internal中







enum：另外我在服务下加了enum枚举目录，因为其他rpc服务或者api服务会调用这个枚举去比对，我就放在internal外部

![1610623135108](/Users/seven/Desktop/go-zero文章/images/五/1610623135108.jpg)

