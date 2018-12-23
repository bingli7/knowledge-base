#  1. 摘要

Traefik是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。 它支持多种后台 (Kubernetes, Docker, Swarm, Marathon, Mesos, Consul, Etcd, Zookeeper等)。

Traefik支持丰富的annotations配置，可配置众多出色的特性，例如：自动熔断、负载均衡策略、黑名单、白名单。所以Traefik对于微服务来说简直就是一神器。

利用Traefik，并结合京东云Kubernetes集群及其他云服务（RDS，NAS，OSS，块存储等），可快速构建弹性扩展的微服务集群。

![image](https://github.com/bingli7/knowledge-base/blob/master/image/traefik/arch.png?raw=true)


本文大致步骤如下：
1. Kubernetes权限配置（RBAC）
2. Traefik部署
3. 创建三个实例服务
4. 生成Ingress规则，并通过PATH测试通过Traefik访问各个服务
5. Traefik配置域名及TLS证书，并实现HTTP重定向到HTTS。


> 本文部署Traefik使用到的Yaml文件均基于Traefik官方实例，并为适配京东云Kubernetes集群做了相关修改：
> 
> https://github.com/containous/traefik/tree/master/examples/k8s

# 2. 京东云Kubernetes集群

京东云Kubernetes整合京东云虚拟化、存储和网络能力，提供高性能可伸缩的容器应用管理能力，简化集群的搭建和扩容等工作，让用户专注于容器化的应用的开发与管理。

用户可以在京东云创建一个安全高可用的 Kubernetes 集群，并由京东云完全托管 Kubernetes 服务，并保证集群的稳定性和可靠性。让用户可以方便地在京东云上使用 Kubernetes 管理容器应用。


京东云Kubernetes集群：https://www.jdcloud.com/cn/products/jcs-for-kubernetes


# 3. Ingress边界路由

虽然Kubernetes集群内部署的pod、server都有自己的IP，但是却无法提供外网访问，虽然我们可以通过监听NodePort的方式暴露服务，但是这种方式并不灵活，生产环境也不建议使用。

Ingresss是k8s集群中的一个API资源对象，扮演边缘路由器(edge router)的角色，也可以理解为集群防火墙、集群网关，我们可以自定义路由规则来转发、管理、暴露服务(一组pod)，非常灵活，生产环境建议使用这种方式。


## 3.1 什么是Ingress？

在Kubernetes中，service和Pod的IP地址仅可以在集群网络内部使用，对于集群外的应用是不可见的。为了使外部的应用能够访问集群内的服务，在Kubernetes中可以通过NodePort和LoadBalancer这两种类型的service，或者使用**Ingress**。

Ingress本质是通过http代理服务器将外部的http请求转发到集群内部的后端服务。通过Ingress，外部应用访问群集内容服务的过程如下所示：

![image](https://www.kubernetes.org.cn/img/2018/05/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180529154927.png)


**Ingress 就是为进入集群的请求提供路由规则的集合。**

Ingress 可以给 service 提供集群外部访问的URL、负载均衡、SSL终止、HTTP路由等。为了配置这些 Ingress 规则，集群管理员需要部署一个 Ingress controller，它监听 Ingress 和 service 的变化，并根据规则配置负载均衡并提供访问入口。


# 4. Traefik是什么？

![image](https://github.com/containous/traefik/blob/master/docs/img/traefik.logo.png?raw=true)

Traefik在Github上有19K星星：
> https://github.com/containous/traefik

**Traefik is a modern HTTP reverse proxy and load balancer designed for deploying microservices.**

Traefik是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。

Traefik是一个用Golang开发的轻量级的Http反向代理和负载均衡器，虽然相比于Nginx，它是后起之秀，但是它天然拥抱kubernetes，直接与集群k8s的Api Server通信，反应非常迅速，同时还提供了友好的控制面板和监控界面，不仅可以方便地查看Traefik根据Ingress生成的路由配置信息，还可以查看统计的一些性能指标数据，如：总响应时间、平均响应时间、不同的响应码返回的总次数等。

不仅如此，Traefik还支持丰富的annotations配置，可配置众多出色的特性，例如：自动熔断、负载均衡策略、黑名单、白名单。所以Traefik对于微服务来说简直就是一神器。


Traefik User Guide for Kubernetes：
> https://docs.traefik.io/user-guide/kubernetes/

# 5. 前置条件

**5.1 创建京东云Kubernetes集群**

创建Kubernetes集群请参考：https://docs.jdcloud.com/cn/jcs-for-kubernetes/create-to-cluster


**5.2 Kubernetes客户端配置**

集群创建完成后，需要配置kubectl客户端以连接Kubernetes集群。请参考：https://docs.jdcloud.com/cn/jcs-for-kubernetes/connect-to-cluster


# 6. Traefik部署

## 6.1 权限配置

创建响应的ClusterRole和ClusterRoleBinding，以赋予traefik足够的权限。

Yaml文件如下：

```
$ cat traefik-rbac.yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
```

开始创建：

```
$ kubectl create -f traefik-rbac.yaml 
clusterrole "traefik-ingress-controller" created
clusterrolebinding "traefik-ingress-controller" created
```

创建成功：

```
$ kubectl get clusterrole -n kube-system | grep traefik
traefik-ingress-controller                                             25s

$ kubectl get clusterrolebinding -n kube-system | grep traefik
traefik-ingress-controller                             35s
```


## 6.2 部署Traefik

本文选择使用Deployment部署Traefik。除此之外，Traefik还提供了DaemonSet的部署方式：

https://github.com/containous/traefik/blob/master/examples/k8s/traefik-ds.yaml

Traefik的80端口为接收HTTP请求，8080端口为dashboard访问端口；通过LoadBalancer类型的Service创建京东云负载均衡SLB，来作为K8S集群的统一入口。

Yaml文件如下：

```
$ cat traefik-deployment.yaml 
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
        - name: admin
          containerPort: 8080
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
  type: LoadBalancer
```

开始创建：

```
$ kubectl create -f traefik-deployment.yaml 
serviceaccount "traefik-ingress-controller" created
deployment "traefik-ingress-controller" created
service "traefik-ingress-service" created
```

Pod正常运行：

```
$ kubectl get pod -n kube-system | grep traefik
traefik-ingress-controller-668679b744-jvmbg   1/1       Running   0          57s
```

查看Pod日志：

```
$ kubectl logs traefik-ingress-controller-668679b744-jvmbg -n kube-system
time="2018-12-15T16:58:49Z" level=info msg="Traefik version v1.7.6 built on 2018-12-14_06:43:37AM"
time="2018-12-15T16:58:49Z" level=info msg="\nStats collection is disabled.\nHelp us improve Traefik by turning this feature on :)\nMore details on: https://docs.traefik.io/basics/#collected-data\n"
time="2018-12-15T16:58:49Z" level=info msg="Preparing server http &{Address::80 TLS:<nil> Redirect:<nil> Auth:<nil> WhitelistSourceRange:[] WhiteList:<nil> Compress:false ProxyProtocol:<nil> ForwardedHeaders:0xc0005f9e20} with readTimeout=0s writeTimeout=0s idleTimeout=3m0s"
time="2018-12-15T16:58:49Z" level=info msg="Preparing server traefik &{Address::8080 TLS:<nil> Redirect:<nil> Auth:<nil> WhitelistSourceRange:[] WhiteList:<nil> Compress:false ProxyProtocol:<nil> ForwardedHeaders:0xc0005f9e40} with readTimeout=0s writeTimeout=0s idleTimeout=3m0s"
time="2018-12-15T16:58:49Z" level=info msg="Starting provider configuration.ProviderAggregator {}"
time="2018-12-15T16:58:49Z" level=info msg="Starting server on :80"
time="2018-12-15T16:58:49Z" level=info msg="Starting server on :8080"
time="2018-12-15T16:58:49Z" level=info msg="Starting provider *kubernetes.Provider {\"Watch\":true,\"Filename\":\"\",\"Constraints\":[],\"Trace\":false,\"TemplateVersion\":0,\"DebugLogGeneratedTemplate\":false,\"Endpoint\":\"\",\"Token\":\"\",\"CertAuthFilePath\":\"\",\"DisablePassHostHeaders\":false,\"EnablePassTLSCert\":false,\"Namespaces\":null,\"LabelSelector\":\"\",\"IngressClass\":\"\",\"IngressEndpoint\":null}"
time="2018-12-15T16:58:49Z" level=info msg="ingress label selector is: \"\""
time="2018-12-15T16:58:49Z" level=info msg="Creating in-cluster Provider client"
time="2018-12-15T16:58:50Z" level=info msg="Server configuration reloaded on :80"
time="2018-12-15T16:58:50Z" level=info msg="Server configuration reloaded on :8080"
```

查看Traefik对应的SLB Service：

```
$ kubectl get svc/traefik-ingress-service -n kube-system
NAME                      TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                       AGE
traefik-ingress-service   LoadBalancer   10.0.58.175   114.67.95.167   80:30331/TCP,8080:30232/TCP   2m
```

京东云负载均衡的公网IP为114.67.95.167。

如果需要通过域名访问K8S内的服务，则可以通过将域名解析至该公网IP。

此时，通过“公网IP:8080”便可以访问Traefik的Dashboard。

## 6.3 Traefik使用示例

### 6.3.1 创建服务

创建3个deployment对外提供HTTP服务，分别名为：stilton、cheddar、wensleydale。


```
$ cat cheese-deployments.yaml
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: stilton
  labels:
    app: cheese
    cheese: stilton
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cheese
      task: stilton
  template:
    metadata:
      labels:
        app: cheese
        task: stilton
        version: v0.0.1
    spec:
      containers:
      - name: cheese
        image: errm/cheese:stilton
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 50Mi
        ports:
        - containerPort: 80
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: cheddar
  labels:
    app: cheese
    cheese: cheddar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cheese
      task: cheddar
  template:
    metadata:
      labels:
        app: cheese
        task: cheddar
        version: v0.0.1
    spec:
      containers:
      - name: cheese
        image: errm/cheese:cheddar
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 50Mi
        ports:
        - containerPort: 80
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: wensleydale
  labels:
    app: cheese
    cheese: wensleydale
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cheese
      task: wensleydale
  template:
    metadata:
      labels:
        app: cheese
        task: wensleydale
        version: v0.0.1
    spec:
      containers:
      - name: cheese
        image: errm/cheese:wensleydale
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
          limits:
            cpu: 100m
            memory: 50Mi
        ports:
        - containerPort: 80
```


```
$ kubectl create -f cheese-deployments.yaml 
deployment "stilton" created
deployment "cheddar" created
deployment "wensleydale" created
```

对应的service：

```
$ cat cheese-services.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: stilton
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: cheese
    task: stilton
---
apiVersion: v1
kind: Service
metadata:
  name: cheddar
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: cheese
    task: cheddar
---
apiVersion: v1
kind: Service
metadata:
  name: wensleydale
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: cheese
    task: wensleydale
```

```
$ kubectl create -f cheese-services.yaml 
service "stilton" created
service "cheddar" created
service "wensleydale" created
```

### 6.3.2 创建Ingress

Ingress Yaml文件如下：
```
$ cat my-cheeses-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cheeses
  annotations:
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - host: www.<your-domain-name>.com 
    http:
      paths:
      - path: /stilton
        backend:
          serviceName: stilton
          servicePort: http
      - path: /cheddar
        backend:
          serviceName: cheddar
          servicePort: http
      - path: /wensleydale
        backend:
          serviceName: wensleydale
          servicePort: http
```

创建Ingress:
```
$ kubectl create -f my-cheeses-ingress.yaml 
ingress "cheeses" created
```

创建成功：
```
$ kubectl describe ingress/cheeses
Name:             cheeses
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host             Path  Backends
  ----             ----  --------
  www.<your-domain-name>.com  
                   /stilton       stilton:http (<none>)
                   /cheddar       cheddar:http (<none>)
                   /wensleydale   wensleydale:http (<none>)
Annotations:
Events:  <none>
```

### 6.3.3 访问服务

直接通过ELB IP+PATH访问：

```
$ curl 114.67.95.167/stilton
404 page not found
```

访问失败，因为ingress规则里指定了host。

请求Header中指定Host：

```
$ curl -H "Host:www.<your-domain-name>.com" 114.67.95.167/stilton
<html>
  <head>
    <style>
      html { 
        background: url(./bg.png) no-repeat center center fixed; 
        -webkit-background-size: cover;
        -moz-background-size: cover;
        -o-background-size: cover;
        background-size: cover;
      }

      h1 {
        font-family: Arial, Helvetica, sans-serif;
        background: rgba(187, 187, 187, 0.5);
        width: 3em;
        padding: 0.5em 1em;
        margin: 1em;
      }
    </style>
  </head>
  <body>
    <h1>Stilton</h1>
  </body>
</html>

```
访问成功。

但是由于域名未备案，这种方式会被京东云拦截。

两种方式：  
一、Ingress里移除指定host  
二、注册域名，并绑定证书及私钥。

### 6.3.4 从Ingress中移除host

将host字段注释掉：
```
$ cat my-cheeses-ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cheeses
  annotations:
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
#  - host: www.<your-domain-name>.com 
  - http:
      paths:
      - path: /stilton
        backend:
          serviceName: stilton
          servicePort: http
      - path: /cheddar
        backend:
          serviceName: cheddar
          servicePort: http
      - path: /wensleydale
        backend:
          serviceName: wensleydale
          servicePort: http
```

重建ingress：
```
$ kubectl replace -f my-cheeses-ingress.yaml 
ingress "cheeses" replaced
$ kubectl get ingress
NAME      HOSTS     ADDRESS   PORTS     AGE
cheeses   *                   80        18m
```
ingress更新成功，通过公网IP+PATH访问。

stilton服务：
```
$ curl -I 114.67.95.167/stilton
HTTP/1.1 200 OK
Accept-Ranges: bytes
Content-Length: 517
Content-Type: text/html
Date: Thu, 20 Dec 2018 06:19:15 GMT
Etag: "5784f6c9-205"
Last-Modified: Tue, 12 Jul 2016 13:55:21 GMT
Server: nginx/1.11.1
```
cheddar服务：
```
$ curl -I 114.67.95.167/cheddar
HTTP/1.1 200 OK
Accept-Ranges: bytes
Content-Length: 517
Content-Type: text/html
Date: Thu, 20 Dec 2018 06:19:54 GMT
Etag: "5784f6e1-205"
Last-Modified: Tue, 12 Jul 2016 13:55:45 GMT
Server: nginx/1.11.1
```
wensleydale服务：
```
$ curl -I 114.67.95.167/wensleydale
HTTP/1.1 200 OK
Accept-Ranges: bytes
Content-Length: 521
Content-Type: text/html
Date: Thu, 20 Dec 2018 06:20:00 GMT
Etag: "5784f6fb-209"
Last-Modified: Tue, 12 Jul 2016 13:56:11 GMT
Server: nginx/1.11.1
```

三个服务均可通过<ELB-public-ip>/<service-name>正常访问。

### 6.3.5 配置域名及证书

申请域名：<your-domain-name>.com，并在京东云上备案，并解析到SLB公网IP：114.67.95.167


证书和私钥：
```
$ ll *.pem
-rw-r--r-- 1 pmo_jd_a pmo_jd_a 3554 Dec 20 16:04 fullchain.pem
-rw------- 1 pmo_jd_a pmo_jd_a 1708 Dec 20 16:04 privkey.pem
```

创建secret保存证书和私钥：
```
$ kubectl create secret generic traefik-cert --from-file=fullchain.pem --from-file=privkey.pem -n kube-system
secret "traefik-cert" created
```

Traefik配置文件(HTTP访问重定向到HTTPS，证书及私钥存放在/ssl/目录下，需要secret挂载到该目录以供traefik读取)：
```
# cat traefik.toml
defaultEntryPoints = ["http","https"]
[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
      entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      CertFile = "/ssl/fullchain.pem"
      KeyFile = "/ssl/privkey.pem"
```

创建configmap用于保存配置文件traefik.toml：
```
$ kubectl create configmap traefik-conf --from-file=traefik.toml -n kube-system
configmap "traefik-conf" created
```

需要重新部署traefik，新的yaml文件如下：

```
$ cat traefik-deployment-new.yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
        - name: admin
          containerPort: 8080
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --configfile=/config/traefik.toml
        volumeMounts:
        - mountPath: "/ssl"
          name: "ssl"
        - mountPath: "/config"
          name: "config"
      volumes:
      - name: ssl
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-conf
```

重新部署：
```
$ kubectl replace -f traefik-deployment-new.yaml 
deployment "traefik-ingress-controller" replaced
$ kubectl get pod -n kube-system | grep traefik
traefik-ingress-controller-668679b744-jvmbg   0/1       Terminating         0          4d
traefik-ingress-controller-7d6cd769c9-2p57t   0/1       ContainerCreating   0          3s
$ kubectl get pod -n kube-system | grep traefik
traefik-ingress-controller-7d6cd769c9-2p57t   1/1       Running   0          19s
```
重新部署的pod正常running。

查看pod日志：

```
$ kubectl logs traefik-ingress-controller-7d6cd769c9-2p57t -n kube-system
time="2018-12-20T09:29:30Z" level=info msg="Preparing server traefik &{Address::8080 TLS:<nil> Redirect:<nil> Auth:<nil> WhitelistSourceRange:[] WhiteList:<nil> Compress:false ProxyProtocol:<nil> ForwardedHeaders:0xc00072dbc0} with readTimeout=0s writeTimeout=0s idleTimeout=3m0s"
time="2018-12-20T09:29:30Z" level=info msg="Preparing server http &{Address::80 TLS:<nil> Redirect:0xc00059de40 Auth:<nil> WhitelistSourceRange:[] WhiteList:<nil> Compress:false ProxyProtocol:<nil> ForwardedHeaders:0xc00072db80} with readTimeout=0s writeTimeout=0s idleTimeout=3m0s"
time="2018-12-20T09:29:30Z" level=info msg="Preparing server https &{Address::443 TLS:0xc000216c60 Redirect:<nil> Auth:<nil> WhitelistSourceRange:[] WhiteList:<nil> Compress:false ProxyProtocol:<nil> ForwardedHeaders:0xc00072dba0} with readTimeout=0s writeTimeout=0s idleTimeout=3m0s"
time="2018-12-20T09:29:30Z" level=info msg="Starting provider configuration.ProviderAggregator {}"
time="2018-12-20T09:29:30Z" level=info msg="Starting server on :8080"
time="2018-12-20T09:29:30Z" level=info msg="Starting server on :80"
time="2018-12-20T09:29:30Z" level=info msg="Starting server on :443"
time="2018-12-20T09:29:30Z" level=info msg="Starting provider *kubernetes.Provider {\"Watch\":true,\"Filename\":\"\",\"Constraints\":[],\"Trace\":false,\"TemplateVersion\":0,\"DebugLogGeneratedTemplate\":false,\"Endpoint\":\"\",\"Token\":\"\",\"CertAuthFilePath\":\"\",\"DisablePassHostHeaders\":false,\"EnablePassTLSCert\":false,\"Namespaces\":null,\"LabelSelector\":\"\",\"IngressClass\":\"\",\"IngressEndpoint\":null}"
time="2018-12-20T09:29:30Z" level=info msg="ingress label selector is: \"\""
time="2018-12-20T09:29:30Z" level=info msg="Creating in-cluster Provider client"
time="2018-12-20T09:29:30Z" level=info msg="Server configuration reloaded on :8080"
time="2018-12-20T09:29:30Z" level=info msg="Server configuration reloaded on :80"
time="2018-12-20T09:29:30Z" level=info msg="Server configuration reloaded on :443"
```


更新ingress：

```
$ cat my-cheeses-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cheeses
  annotations:
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - host: www.<your-domain-name>.com 
    http:
      paths:
      - path: /stilton
        backend:
          serviceName: stilton
          servicePort: http
      - path: /cheddar
        backend:
          serviceName: cheddar
          servicePort: http
      - path: /wensleydale
        backend:
          serviceName: wensleydale
          servicePort: http
```

重建ingress：
```
$ kubectl replace -f my-cheeses-ingress.yaml
```

更新traefik service，开放443端口：

```
$ cat traefik-service.yaml 
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
    - protocol: TCP
      port: 443
      name: tls
  type: LoadBalancer
```

应用service更新：
```
$ kubectl apply -f traefik-service.yaml -n kube-system
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
service "traefik-ingress-service" configured
```

HTTPS访问：https://www.your-domain-name.com/stilton

以及HTTP：http://www.your-domain-name.com/stilton

![image](https://github.com/bingli7/knowledge-base/blob/master/image/traefik/https_stilton.png?raw=true)

![image](https://github.com/bingli7/knowledge-base/blob/master/image/traefik/certification.png?raw=true)

均可正常访问，且HTTP访问会被重定向到HTTPS。

# 7. 总结

本文仅测试了traefik的路由分发，当然traefik的功能远远不止于此，其大致特性如下：
- 它非常快~~~
- 无需安装其他依赖，通过Go语言编写的单一可执行文件
- 支持 Rest API
- 多种后台支持：Docker, Swarm, Kubernetes, Marathon, Mesos, Consul, Etcd, 并且还会更多
- 后台监控, 可以监听后台变化进而自动化应用新的配置文件设置
- 配置文件热更新。无需重启进程
- 正常结束http连接
- 后端断路器
- 轮询，rebalancer 负载均衡
- Rest Metrics
- 支持最小化 官方 docker 镜像
- 后台支持SSL
- 前台支持SSL（包括SNI）
- 清爽的AngularJS前端页面
- 支持Websocket
- 支持HTTP/2
- 网络错误重试
- 支持Let’s Encrypt (自动更新HTTPS证书)
- 高可用集群模式

以上。

