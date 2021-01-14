以往我们手写Dockerfile、k8s的yaml都比较烦，尽管k8s有helm还是不太好用（相对于goctl生成的yaml，哈哈，谁用谁知道）

Dockerfile生成，进入api/v1/usercenter目录下，执行 

```shell
$ goctl docker -go usercenter.go
```

![1610625460373](/Users/seven/Desktop/go-zero文章/images/九/1610625460373.jpg)

```dockerfile
FROM golang:alpine AS builder

LABEL stage=gobuilder

ENV CGO_ENABLED 0
ENV GOOS linux

WORKDIR /build/zero

ADD go.mod .
ADD go.sum .
RUN go mod download
COPY . .
COPY gateway/api/v1/usercenter/etc /app/etc
RUN go build -ldflags="-s -w" -o /app/usercenter gateway/api/v1/usercenter/usercenter.go


FROM alpine

RUN apk update --no-cache && apk add --no-cache ca-certificates tzdata
ENV TZ Asia/Shanghai

WORKDIR /app
COPY --from=builder /app/usercenter /app/usercenter
COPY --from=builder /app/etc /app/etc

CMD ["./usercenter", "-f", "etc/usercenter-api.yaml"]
```





生成k8s

```shell
 goctl kube deploy -name user-api -namespace fishtwo -image user:v1 -o user.yaml -port 2233
```

user.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-api
  namespace: fishtwo
  labels:
    app: user-api
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: user-api
  template:
    metadata:
      labels:
        app: user-api
    spec:
      containers:
      - name: user-api
        image: user:v1
        lifecycle:
          preStop:
            exec:
              command: ["sh","-c","sleep 5"]
        ports:
        - containerPort: 2233
        readinessProbe:
          tcpSocket:
            port: 2233
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 2233
          initialDelaySeconds: 15
          periodSeconds: 20
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1024Mi
        volumeMounts:
        - name: timezone
          mountPath: /etc/localtime
      volumes:
        - name: timezone
          hostPath:
            path: /usr/share/zoneinfo/Asia/Shanghai

---

apiVersion: v1
kind: Service
metadata:
  name: user-api-svc
  namespace: fishtwo
spec:
  ports:
    - port: 2233
  selector:
    app: user-api

---

apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: user-api-hpa-c
  namespace: fishtwo
  labels:
    app: user-api-hpa-c
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80

---

apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: user-api-hpa-m
  namespace: fishtwo
  labels:
    app: user-api-hpa-m
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80

```







至于如何部署，前面架构篇已经有详细流程图了，另外这里在附上go-zero官方给出的教程，使用起来，爽的一比啊～～

Dockerfile : https://gocn.vip/topics/11370

K8S YAML: https://gocn.vip/topics/11380

k8s部署:https://mp.weixin.qq.com/s/jMsL464stSRctj2OLjOZBg 

【注】：k8s部署那篇文章，有的地方是写错了，但是是公众号没办法更改，都是些小问题，自己使用时候有问题时根据报错自己调整下就好了，当时我想吧这个部署流程的坑都记录下来，但是没记～～哈哈～～嗯嗯～～ 过了就过了 ，下次在记吧～～哈哈





