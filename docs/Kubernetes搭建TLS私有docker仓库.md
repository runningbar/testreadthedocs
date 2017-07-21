---
title: Kubernetes搭建TLS私有docker仓库
date: 2017-06-07 19:10:30
categories: Kubernetes
---

> 以下流程在`minikube`中完全跑通

## 完成效果

* 可以在kubernetes集群中，搭建自己的`docker registry`
* `registry`支持TLS验证，并且TLS证书支持IP验证【默认只支持域名验证】
* `registry`不仅可以被集群中`pod`访问，而且可以被集群外部访问。
* `registry`使用的存储基于NFS，这就意味着你可以将镜像存在网络中其它主机上。

## 完整流程

### 创建PV

仓库的首要任务就是存储数据。我们需要决定将数据存储在哪里，可以使用Kubernetes的`PersistentVolume`对象实现持久化。

```
# kube-registry-pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
        name: kube-system-kube-registry-pv
        labels:
                kubernetes.io/cluster-service: "true"
spec:
        capacity:
                storage: 23Gi
        accessModes:
                - ReadWriteOnce
        storageClassName: nfs-registry
        nfs:
                path: "/home/mine/nfs"
                server: 192.168.1.100
```

这里面使用nfs作为持久化挂载，关于如何搭建NFS服务器，可以查看我的这篇文章：[Linux NFS使用](http://www.huamo.online/2017/06/05/Ubuntu-NFS%E4%BD%BF%E7%94%A8/) 。

### 创建PVC来绑定PV

PV持久卷创建完毕后，我们可以使用`PersistentVolumeClaim`来绑定这个PV。

```
# kube-registry-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
        name: kube-registry-pvc
        namespace: kube-system
        labels:
                kubernetes.io/cluster-service: "true"
spec:
        accessModes:
                - ReadWriteOnce
        storageClassName: nfs-registry
        resources:
                requests:
                        storage: 23Gi

```

### 创建docker registry

现在我们可以创建一个Docker仓库了。

```
# kube-registry-rc.yaml

apiVersion: v1
kind: ReplicationController
metadata:
    name: kube-registry-v0
    namespace: kube-system
    labels:
        k8s-app: kube-registry
        version: v0
        #kubernetes.io/cluster-service: "true"
spec:
    replicas: 1
    selector:
        k8s-app: kube-registry
        version: v0
    template:
        metadata:
            labels:
                k8s-app: kube-registry
                version: v0
                #kubernetes.io/cluster-service: "true"
        spec:
            containers:
            - name: registry
              image: registry:2
              resources:
                limits:
                    cpu: 100m
                    memory: 100Mi
              env:
              - name: REGISTRY_HTTP_ADDR
                value: :5000
              - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
                value: /var/lib/registry
              - name: REGISTRY_HTTP_TLS_CERTIFICATE
                value: /certs/server.crt
              - name: REGISTRY_HTTP_TLS_KEY
                value: /certs/server.key
              volumeMounts:
              - name: image-store
                mountPath: /var/lib/registry
              - name: cert-dir
                mountPath: /certs
              ports:
              - containerPort: 5000
                hostPort: 5000
                name: registry
                protocol: TCP
            volumes:
            - name: image-store
              persistentVolumeClaim:
                claimName: kube-registry-pvc
            - name: cert-dir
              secret:
                secretName: registry-tls-secret
```

这里面使用了`hostPort: 5000`将Pod中的仓库端口映射到minikube的`5000`端口中，以便集群外部也能访问这个docker registry。

并且，我们通过使用名叫`registry-tls-secret`的`Secret`资源，为这个registry pod提供了TLS必要的`server.crt`和`server.key`。这个`Secrets`以及证书相关，会在下文介绍。

### 创建Service暴露Pod服务

我们可以将registry Pod作为一个`Service`暴露出来。

```
# kube-registry-svc.yaml
apiVersion: v1
kind: Service
metadata:
    name: kube-registry
    namespace: kube-system
    labels:
        k8s-app: kube-registry
        #kubernetes.io/cluster-service: "true"
        kubernetes.io/name: "KubeRegistry"
spec:
    selector:
        k8s-app: kube-registry
    #type: NodePort
    ports:
    - name: registry
      port: 5000
      #nodePort: 30001
      protocol: TCP
```

OK，目前为止，我们已经建立了docker registry的相关资源，接下来就是准备TLS支持的相关工作。

### 生成服务端crt证书和key私钥

这一段走的时候不太顺利，尝试了很多方法，也解决了很多问题，大致包括：

* TLS证书需要支持IP验证。默认情况下证书只支持域名校验，否则会报`IP SANs`错误`cannot validate certificate for 192.168.1.100 because it doesn't contain any IP SANs`。
* 不能使用自签名证书。首先必须要自己生成一个CA crt证书，然后用这个CA来签名服务端证书，然后把该CA证书放在docker的`custom root CA`目录下，如果采用自签名证书，或者不提供CA，则会报错`x509: certificate signed by unknown authority`。

如前所述，我们将registry服务映射到了minikube的`5000`端口上，假设minikube的IP地址为`192.168.99.100`，根据这些信息我们来生成`192.168.99.100:5000`服务端TLS证书。

> 当然了，如果你是土豪，可以直接到公网CA上直接买证书，就可以跳过这一段辛酸的经验

#### 解决`IP SANs`问题

为了让证书支持IP验证，我们首先需要定制一个自己的openssl模板文件，创建文件`openssl.cnf`，内容如下：

```
[ req ]
#default_bits           = 2048
#default_md             = sha256
#default_keyfile        = privkey.pem
distinguished_name      = req_distinguished_name
attributes              = req_attributes
req_extensions          = v3_req

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_min                 = 2
countryName_max                 = 2
stateOrProvinceName             = State or Province Name (full name)
localityName                    = Locality Name (eg, city)
0.organizationName              = Organization Name (eg, company)
organizationalUnitName          = Organizational Unit Name (eg, section)
commonName                      = Common Name (eg, fully qualified host name)
commonName_max                  = 64
emailAddress                    = Email Address
emailAddress_max                = 64

[ req_attributes ]
challengePassword               = A challenge password
challengePassword_min           = 4
challengePassword_max           = 20

[ v3_req ]
subjectAltName = IP:192.168.99.100
```

主要是`subjectAltName = IP:192.168.99.100`这句话，开启了对IP的验证功能。

#### 生成`ca.key`

```
openssl genrsa -out ca.key 2048
```

#### 基于`openssl.cnf`生成`ca.crt`

```
openssl req -x509 -new -nodes -key ca.key -subj "/CN=192.168.99.100:5000" -days 5000 -config openssl.cnf -out ca.crt
```

> 备注：当然，推荐使用域名作为CN，这样的话，就不需要openssl.cnf文件了，只需要在hosts文件中做域名到IP的映射就行，如果你有公网域名，那就更方便了，连hosts也不用改，DNS服务商已为你做好了一切。

#### 生成`server.key`

```
openssl genrsa -out server.key 2048
```

这个`server.key`就是前面`ReplicationController`配置文件里面提到的key文件。

#### 基于`openssl.cnf`生成`server.csr`

```
openssl req -new -key server.key -subj "/CN=192.168.99.100:5000" -config openssl.cnf -out server.csr
```

#### 生成`server.crt`

```
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 5000 -extensions v3_req -extfile openssl.cnf
```

这个命令同时还会生成`ca.srl`文件。这个`server.crt`就是前面`ReplicationController`配置文件里面提到的crt文件。

到这儿，TLS所需要的证书和私钥已经全部准备完毕。

### 将server.key和server.crt打包到Secrets中

准备好了`server.key`和`server.crt`，需要提供给registry pod，这样才能提供TLS支持，这里使用kubernetes的`Secrets`资源来提供配置。

```
kubectl --namespace=kube-system create secret generic registry-tls-secret --from-file=server.crt=./server.crt --from-file=server.key=./server.key
```

这个`Secret`资源名称即为`registry-tls-secret`，正好被之前的`ReplicationController`所引用。

### 将所有准备工作串起来

到这里，所有的准备资源都已就绪，现在需要把它们整合起来并运行。首先看下当前的目录结构：

```
.
├── ca.crt
├── ca.key
├── ca.srl
├── kube-registry-pvc.yaml
├── kube-registry-pv.yaml
├── kube-registry-rc.yaml
├── kube-registry-svc.yaml
├── openssl.cnf
├── server.crt
├── server.csr
└── server.key
```

创建文件`start.sh`，这样便于随时自动化启动运行：

```
kubectl --namespace=kube-system create secret generic registry-tls-secret --from-file=server.crt=./server.crt --from-file=server.key=./server.key
kubectl create -f kube-registry-pv.yaml
kubectl create -f kube-registry-pvc.yaml
kubectl create -f kube-registry-rc.yaml
kubectl create -f kube-registry-svc.yaml
```

创建文件`delete.sh`，便于自动化关闭服务：

```
kubectl --namespace=kube-system delete secrets registry-tls-secret
kubectl delete -f kube-registry-pv.yaml
kubectl delete -f kube-registry-pvc.yaml
kubectl delete -f kube-registry-rc.yaml
kubectl delete -f kube-registry-svc.yaml
```

将一切运行起来：

```
sh ./delete.sh /*删除所有相关资源以初始化*/
sh ./start.sh /*运行支持tls的docker registry*/
kubectl --namespace=kube-system get all /*查看所有相关资源是否都正常running*/

/*有可能会因为在墙内，导致相关container镜像下载失败，进而不能正常运行，需要自行翻墙*/
```

### 验证这一切跑通

我们可以在能访问`192.168.99.100`的任一台机器上验证这一切，在验证之前，还有一项准备工作要做，否则docker会提示你证书无效：`x509: certificate signed by unknown authority`

#### 向验证机器的docker提供CA证书：

```
sudo mkdir -p /etc/docker/certs.d/192.168.99.100:5000
/*如果之前使用域名作为CN，那这里就用域名作为最后一个目录名字*/

/*将之前生成的ca.crt放在docker custom root ca目录下*/
cp ./ca.crt /etc/docker/certs.d/192.168.99.100:5000/
```

#### 验证

```
docker pull busybox
docker tag busybox 192.168.99.100:5000/busybox
docker push 192.168.99.100:5000/busybox
docker pull 192.168.99.100:5000/busybox
```


## 参考文章

* [Private Docker Registry in Kubernetes](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/registry)
* [Enable TLS for Kube-Registry](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/registry/tls/README.md)
* [Https原理以及服务端和客户端golang的基本实现](http://www.tuicool.com/articles/IZza6jA)
* [How are SSL certificate server names resolved/Can I add alternative names using keytool?](https://stackoverflow.com/questions/8443081/how-are-ssl-certificate-server-names-resolved-can-i-add-alternative-names-using/8444863#8444863)

