---
layout: post
title: "SpringBoot on Kubernetes"
date: 2018-03-24 16:40
comments: true
categories: 
---

本文将从创建一个 SpringBoot 的应用开始，详细讲述如何将一个 SpringBoot 的应用部署到本地的 Kubernetes 集群上面去。

## 创建 SpringBoot 应用
首先，我们需要创建一个 SpringBoot 的应用，可以到 [http://start.spring.io/](http://start.spring.io/) 创建一个，在创建的时候，我们需要选择一个 Web 的依赖，以方便部署到 Kubernetes 之后可以看到效果。

创建完成之后，可以修改一下 SpringBoot 应用的 `main` 函数所在的类，让它成为一个 Controller：

```java
@SpringBootApplication
@Controller
public class WebDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebDemoApplication.class, args);
    }

    @RequestMapping("/hello")
    @ResponseBody
    public String hello() {
        return "Hello, Kubernetes!";
    }
}
```

这样，在本地启动这个 SpringBoot 的应用之后，如果我们访问 `http://localhost:8080/hello` 的话，就可以看到 `Hello, Kubernetes!` 这句话。

## 创建一个 Dockerfile
为了能够将这个 SpringBoot 的应用部署到 Kubernetes 里面去，我们需要创建一个 Dockerfile，将它打成一个 Docker 镜像：

```
FROM openjdk:8-jdk-alpine
ARG JAR_FILE
ADD ${JAR_FILE} app.jar
ENTRYPOINT [ "java", "-jar", "/app.jar"]
```

上面的 Dockerfile 是一个非常简单的 Dockerfile，只是将 SpringBoot 应用打包后的 uber-jar 拷贝到容器里面去，然后运行这个 jar 包。有了这个 Dockerfile 之后，我们就可以在本地把 Docker 镜像打包出来了：

```
docker build --build-arg JAR_FILE=./target/web-demo-0.0.1-SNAPSHOT.jar . -t springboot-demo
```

然后需要注意的是，这样打出来的镜像是在本地的，没有办法被 minikube 找到，所以，要么将这个镜像放到一个中央的镜像仓库上，要么我们使用 minikube 的 docker daemon 来打镜像，这样 minikube 就可以找到这个镜像。

所以，你首先需要在本地将 minikube 安装上去，具体可以看官方的[安装教程](https://kubernetes.io/docs/tasks/tools/install-minikube/)。安装完成之后，先运行：

```
minikube start
```

来将 minikube 启动起来，然后可以运行

```
eval $(minikube docker-env)
```

将 docker daemon 切换成 minikube 的。最后，我们再用上面的 `docker build` 来进行打包，minikube 就可以看到了。

## 将应用部署到 minikube 中去
Docker 镜像都准备好了，现在我们可以将应用部署到 minikube 中去了，首先我们需要创建一个 deployment 对象，这个可以用 yml 文件来描述：

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: springboot-demo-deployment
  labels:
    app: springboot-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springboot-demo
  template:
    metadata:
      labels:
        app: springboot-demo
    spec:  
      containers:
        - name: springboot-demo
          image: springboot-demo
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

上面的 yaml 文件没有什么特别的地方，除了 `imagePullPolicy` 需要指定成 `IfNotPresent`，这样 minikube 才会从本地去找镜像。

有了上面的 yaml 文件之后，我们就可以运行 `kubectl apply -f springboot-demo.yml` 来让 minikube 将我们的 SpringBoot 的应用的集群给创建出来。

## 访问 minikube 中的 SpringBoot 集群
现在我们已经将 SpringBoot 应用部署到了 minikube 中去，那么怎么访问这个集群呢，首先我们需要将端口暴露出来：

```
kubectl expose deployment springboot-demo-deployment --type=NodePort
```

然后运行：

```
minikube service springboot-demo-deployment --url
```

得到访问的 URL。再在得到的 URL 后面加上 `/hello`，就可以看到 `Hello, Kubernetes!` 了。

或者，我们可以直接运行 `curl $(minikube service springboot-demo-deployment --url)/hello` 来访问。

以上就是如何将一个 SpringBoot 的应用部署到 Kubernetes 里面去的全过程。