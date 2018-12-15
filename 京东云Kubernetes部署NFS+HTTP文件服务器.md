# 前置条件

1. 创建京东云Kubernetes集群
2. 在Kubernetes的所有Node上安装nfs-utils包：
```
# yum install -y nfs-utils
```

如未安装该软件包，则无法挂载NFS卷，会出现如下报错：
> Mounting command: systemd-run
> Mounting arguments: --description=Kubernetes transient mount for /var/lib/kubelet/pods/f87afa9e-ffb4-11e8-a450-fa163e2d4725/volumes/kubernetes.io~nfs/my-nfs --scope -- mount -t nfs 10.0.57.235:/ /var/lib/kubelet/pods/f87afa9e-ffb4-11e8-a450-fa163e2d4725/volumes/kubernetes.io~nfs/my-nfs
> Output: Running scope as unit run-7524.scope.
> mount: wrong fs type, bad option, bad superblock on 10.0.57.235:/,
>        missing codepage or helper program, or other error
>        (for several filesystems (e.g. nfs, cifs) you might
>        need a /sbin/mount.<type> helper program)
> 
>        In some cases useful info is found in syslog - try
>        dmesg | tail or so.
>   Warning  FailedMount  13s  kubelet, k8s-node-vmb8nn-culdo9n2n7  MountVolume.SetUp failed for volume "my-nfs" : mount failed: exit status 32

3. 创建新的k8s namespace
```
$ kubectl create namespace nfs
namespace "nfs" created
```

# 创建NFS持久化存储卷

本文使用动态存储方式创建持久化硬盘，用于NFS Server的数据盘。

京东云默认提供三种storageclass：
```
$ kubectl get storageclass
NAME                PROVISIONER
default (default)   kubernetes.io/jdcloud-ebs
jdcloud-hdd         kubernetes.io/jdcloud-ebs
jdcloud-ssd         kubernetes.io/jdcloud-ebs
```
其中，default storageclas即为jdcloud-ssd（京东云SSD云硬盘）：

```
$ kubectl get storageclass default -o yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  creationTimestamp: 2018-12-07T10:12:43Z
  name: default
  resourceVersion: "271"
  selfLink: /apis/storage.k8s.io/v1/storageclasses/default
  uid: a2fd49a8-fa08-11e8-a450-fa163e2d4725
parameters:
  fstype: ext4
  type: ssd
provisioner: kubernetes.io/jdcloud-ebs
```

创建NFS持久化存储卷，Yaml文件如下：
```
$ cat nfs-ssd-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-ssd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: default 
  resources:
    requests:
      storage: 200Gi
```

开始创建PVC：
```
$ kubectl create -f nfs-ssd-pvc.yaml -n nfs
persistentvolumeclaim "nfs-ssd-pvc" created
```

查看动态创建的PV：
```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM             STORAGECLASS   REASON    AGE
pvc-8729cf78-006d-11e9-bbd6-fa163eb6fb82   200Gi      RWO            Delete           Bound     nfs/nfs-ssd-pvc   default                  4m
```

查看PVC，动态创建的PV已绑定至PVC：nfs-ssd-pvc
```
$ kubectl get pvc -n nfs
NAME          STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-ssd-pvc   Bound     pvc-8729cf78-006d-11e9-bbd6-fa163eb6fb82   200Gi      RWO            default        5m
```

下一步使用该存储卷作为NFS Server的数据盘。

# 创建NFS服务器

k8s内部署NFS Server，并挂在持久化卷至/exports目录。

NFS Server的yaml文件如下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nfs-server
spec:
  replicas: 1
  selector:
    matchLabels:
      role: nfs-server
  template:
    metadata:
      labels:
        role: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: bingli7/volume-nfs:0.8
        imagePullPolicy: IfNotPresent
        ports:
          - name: nfs
            containerPort: 2049
          - name: mountd
            containerPort: 20048
          - name: rpcbind
            containerPort: 111
        securityContext:
          privileged: true
        volumeMounts:
          - mountPath: /exports
            name: datadir 
      restartPolicy: Always
      volumes:
        - name: datadir
          persistentVolumeClaim:
            claimName: nfs-ssd-pvc
```

> 注意：privileged: true

创建NFS Server：
```
$ kubectl create -f nfs-server-deployment.yaml -n nfs
deployment "nfs-server" created
```

pod正常运行：
```
$ kubectl get pod -n nfs
NAME                          READY     STATUS    RESTARTS   AGE
nfs-server-85d48fc6db-chbsd   1/1       Running   0          2m
```

进入NFS Pod查看：
```
$ kubectl exec nfs-server-85d48fc6db-chbsd -n nfs -i -t sh
sh-4.2# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          99G  3.6G   91G   4% /
tmpfs           1.9G     0  1.9G   0% /dev
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/vdb        197G   61M  187G   1% /exports
/dev/vda1        99G  3.6G   91G   4% /etc/hosts
shm              64M     0   64M   0% /dev/shm
tmpfs           1.9G   12K  1.9G   1% /run/secrets/kubernetes.io/serviceaccount
sh-4.2# ps aux 
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  11636  1476 ?        Ss   13:33   0:00 /bin/bash /usr/local/bin/run_nfs.sh /exports /
rpc         11  0.0  0.0  64904  1412 ?        Ss   13:33   0:00 /usr/sbin/rpcbind -w
root        14  0.0  0.1  47996  4096 ?        Ss   13:33   0:00 /usr/sbin/rpc.mountd -N 2 -V 3
rpcuser     18  0.0  0.2  50800  7992 ?        Ss   13:33   0:00 /usr/sbin/rpc.statd --no-notify
root        31  0.0  0.0  11768  1664 pts/0    Ss   13:34   0:00 sh
root        36  0.0  0.0   4312   356 ?        S    13:34   0:00 sleep 5
root        37  0.0  0.0  47420  1668 pts/0    R+   13:34   0:00 ps aux
sh-4.2# cat /etc/exports
/exports *(rw,fsid=0,insecure,no_root_squash)
/ *(rw,fsid=0,insecure,no_root_squash)
```

# 创建NFS Server Service

创建Service以提供外部访问。

Yaml文件如下：

```
$ cat nfs-server-service.yaml 
kind: Service
apiVersion: v1
metadata:
  name: nfs-server
spec:
  ports:
    - name: nfs
      port: 2049
    - name: mountd
      port: 20048
    - name: rpcbind
      port: 111
  selector:
    role: nfs-server
```

创建Service：

```
$ kubectl create -f nfs-server-service.yaml -n nfs
service "nfs-server" created
```

创建成功：

```
$ kubectl get svc/nfs-server -n nfs
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
nfs-server   ClusterIP   10.0.59.70   <none>        2049/TCP,20048/TCP,111/TCP   20s

```

# 创建NFS持久化存储

使用NFS创建存储卷。

获取NFS Server的ClusterIP：

```
$ kubectl get svc/nfs-server -n nfs
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
nfs-server   ClusterIP   10.0.59.70   <none>        2049/TCP,20048/TCP,111/TCP   3m
```

并更新NFS Server的IP为10.0.60.112，大小为200Gi，Yaml文件如下：

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-nfs-pv
spec:
  capacity:
    storage: 200Gi 
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.0.59.70
    path: "/"
```

创建PV：

```
$ kubectl create -f nfs-pv.yaml 
persistentvolume "my-nfs-pv" created
```

PV创建成功：

```
$ kubectl get pv my-nfs-pv
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
my-nfs-pv   200Gi      RWX            Retain           Available                                      17s
```


创建PVC，PVC的Yaml文件如下：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 200Gi
```


创建PVC：
```
$ kubectl create -f nfs-pvc.yaml -n nfs
persistentvolumeclaim "my-nfs-pvc" created
```

绑定PV成功：
```
$ kubectl get pvc/my-nfs-pvc -n nfs
NAME         STATUS    VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-nfs-pvc   Bound     my-nfs-pv   200Gi      RWX                           12s
```


# 创建文件服务器

k8s内部署Nginx作为文件服务器，并挂载NFS存储卷。

Yaml文件：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nfs-web
spec:
  replicas: 2
  selector:
    matchLabels:
      role: web-frontend
  template:
    metadata:
      labels:
        role: web-frontend
    spec:
      containers:
      - name: nfs-web
        image: nginx
        imagePullPolicy: IfNotPresent
        resources: {}
        volumeMounts:
            - name: volume-nfs
              mountPath: "/usr/share/nginx/html"
        ports:
          - name: web
            containerPort: 80
      restartPolicy: Always
      volumes:
      - name: volume-nfs
        persistentVolumeClaim:
          claimName: my-nfs-pvc
```


创建nfs-web deployment：

```
$ kubectl create -f nfs-web-deployment.yaml -n nfs
deployment "nfs-web" created
```

运行成功：

```
$ kubectl get pod -n nfs -o wide
NAME                          READY     STATUS    RESTARTS   AGE       IP          NODE
nfs-server-85d48fc6db-chbsd   1/1       Running   0          9m        10.0.0.10   k8s-node-vmk6x8-culdo9n2n7
nfs-web-7b59b5bc7d-9qkgm      1/1       Running   0          27s       10.0.0.6    k8s-node-vmk6x8-culdo9n2n7
nfs-web-7b59b5bc7d-rhlmk      1/1       Running   0          27s       10.0.0.34   k8s-node-vmb8nn-culdo9n2n7
```

一共2个web pod，分别运行在不同的Node上，NFS支持多点挂载ReadWriteMany模式。

# 创建文件服务器Service

创建k8s Service以提供外部访问文件服务器。

Yaml文件：

```
kind: Service
apiVersion: v1
metadata:
  name: nfs-web
spec:
  ports:
    - port: 80
  selector:
    role: web-frontend
  type: LoadBalancer
```

创建service：

```
$ kubectl create -f nfs-web-service.yaml -n nfs
service "nfs-web" created
```

创建成功：

```
$ kubectl get svc/nfs-web -n nfs
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
nfs-web   LoadBalancer   10.0.58.213   114.67.95.166   80:32722/TCP   1m
```

114.67.95.166为京东云负载均衡SLB的IP。

# 访问测试

登录NFS Server：

```
$ kubectl exec nfs-server-85d48fc6db-chbsd -i -t sh -n nfs
sh-4.2#
```

更改index.html文件：

```
sh-4.2# echo "Hello libing" > /exports/index.html
```

创建新的测试文件：

```
sh-4.2# echo "this is a txt file" > /exports/testfile.txt
```

浏览器访问文件服务器地址：http://114.67.95.166
 
> 返回：Hello libing

访问：http://114.67.95.166/testfile.txt
> 出现：this is a txt file

以上。