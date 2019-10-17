# Kubernetes应用服务日志收集

## 方案选择

Kubernetes官方提供了EFK的日志收集解决方案，但是这种方案并不适合所有的业务场景，它本身就有一些局限性，例如：

* 所有日志都必须是out前台输出，真实业务场景中无法保证所有日志都在前台输出
* 只能有一个日志输出文件，而真实业务场景中往往有多个日志输出文件
* 已经有自己的ELK集群且有专人维护，没有必要再在kubernetes上做一个日志收集服务


## 日志收集解决方案
  方案：结合当前服务的场景，单独创建一个日志收集的容器和应用的容器一起运行在同一个pod中；
  优点：低耦合，扩展性强，方便维护和升级；
  缺点：需要对kubernetes的yaml文件进行单独配置，略显繁琐。

  该方案虽然不一定是最优的解决方案，但是在在扩展性、个性化、部署和后期维护方面都能做到均衡，也适合们现在的使用场景，因此选择该方案。

## 构建镜像

* 编写Filebeat Dockerfile文件

```bash
# mkdir filebeat && cd filebeat
# vim Dockerfile   
FROM alpine:3.10

ENV FILEBEAT_VERSION 7.3.0

RUN apk add --no-cache --virtual=build-dependencies curl iputils busybox-extras &&\
        curl -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-${FILEBEAT_VERSION}-linux-x86_64.tar.gz && \
        tar fx filebeat-${FILEBEAT_VERSION}-linux-x86_64.tar.gz && \
        mv filebeat-${FILEBEAT_VERSION}-linux-x86_64 /filebeat && \
        ln -sf /filebeat/filebeat /bin/filebeat && \
        rm -rf filebeat-${FILEBEAT_VERSION}-linux-x86_64.tar.gz && \
        rm -rf fields.yml filebeat.reference.yml LICENSE.txt NOTICE.txt README.md

CMD ["/filebeat/filebeat", "-e", "-c", "/filebeat/filebeat.yml"]
```

* 构建Docker镜像
```bash
# 实际环境，改成自己公司的私有镜像仓库地址
# docker build -t docker.io/filebeat-agent:v7.3.0 .
```

## 测试实例
```bash
# 注意事项: 将tomcat容器的/usr/local/tomcat/logs目录挂载到filebeat容器的/logs目录
# vim filebeat-agent.yaml    
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: tomcat-testing
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: tomcat-testing
    spec:
      containers:
      - name: tomcat
        image: tomcat:8.5
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: tomcat-logs
          mountPath: /usr/local/tomcat/logs
      - name: filebeat
        image: docker.io/filebeat-agent:v7.3.0
        volumeMounts:
        - name: tomcat-logs
          mountPath: /logs
        - name: filebeat-config
          mountPath: /filebeat
      volumes:
      - name: tomcat-logs
        emptyDir: {}
      - name: filebeat-config
        configMap:
          name: filebeat-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.prospectors:
    - input_type: log
      paths:
        - /logs/catalina.*.log
      multiline:
        pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
        negate: true
        match: after

    output.kafka:
      hosts: ["192.168.3.95:9092"]
      topic: 'k8s-filebeat'
      partition.round_robin:
        reachable_only: false
      required_acks: 1
      compression: gzip
      max_retries: 3
      bulk_max_size: 4096
      max_message_bytes: 1000000
# kubectl apply -f filebeat-agent.yaml
```

## 验证测试

待补充......
