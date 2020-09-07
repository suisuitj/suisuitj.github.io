# Static Pod - 静态Pod

1. 静态Pod` （`Static Pod`）是被`kubelet`直接管理的Pod类型，`kubelet`作为`静态Pod`的守护进程，负责监控其状态

2. `kubelet`创建`静态Pod`同时，会为每个`静态Pod`尝试向`API server`请求创建一个`mirror Pod`，这样通过`API server`就可以看到该Pod的状态。

3. `静态Pod`不受`API server`控制和管理，通过`kubectl`删除`静态Pod`不会影响该Pod



通过以下两种`kubelet`参数 启动`静态Pod`：

* `--pod-manifest-path=<file_dir>` ：配置Pod定义文件 (static Pod manifest) 路径

* `--manifest-url=<URL>` ： 配置Pod定义文件URL

`kubelet`周期扫描对应Pod文件或者下载远程Pod文件，维护`静态Pod`状态，能够监控文件发生变化后，自动从新拉取新配置的`静态Pod`

## 应用实例

部署K8S集群时，通过`静态Pod`方式部署API Server

* 确认/添加 kubelet配置参数  `staticPodPath`

```shell
$ service kubelet status

Redirecting to /bin/systemctl status kubelet.service
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2020-09-01 21:07:42 CST; 5 days ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 9200 (kubelet)
    Tasks: 13
   Memory: 101.4M
   CGroup: /system.slice/kubelet.service
           └─9200 /usr/bin/kubelet --allow-privileged=true --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.conf --cert-dir=/etc/kubernetes/pki --cni-conf-dir=/etc/cni/net.d --container-runtime=docker --kubeconfig=/etc/kubernetes/kubelet.conf --config=/etc/kub...

$ cat /usr/lib/systemd/system/kubelet.service | grep config=

ExecStart=/bin/bash -c "exec /usr/bin/kubelet --allow-privileged=true --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.conf --cert-dir=/etc/kubernetes/pki --cni-conf-dir=/etc/cni/net.d --container-runtime=docker --kubeconfig=/etc/kubernetes/kubelet.conf --config /etc/kubelet/kubelet-config.yaml --pod-infra-container-image=tpaas-registry.jdcloud.com/k8s/pause-amd64:3.1 --logtostderr=true --v=2 "

$ cat /etc/kubelet/kubelet-config.yaml | grep staticPodPath

staticPodPath: /etc/kubernetes/manifests
```

如果没有 `staticPodPath` 配置项，手动添加，重启kubelet service即可。

* 将`kube-apiserver.yaml` 复制到 指定节点（通常是master节点） `/etc/kubernetes/manifests` 目录下

```yaml
# kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --default-not-ready-toleration-seconds=360
    - --default-unreachable-toleration-seconds=360
    - --max-mutating-requests-inflight=2000
    - --max-requests-inflight=4000
    - --default-watch-cache-size=200
    - --delete-collection-workers=2
    - --etcd-cafile=/etc/kubernetes/pki/ca.pem
    - --etcd-certfile=/etc/kubernetes/pki/etcd/etcd.pem
    - --etcd-keyfile=/etc/kubernetes/pki/etcd/etcd-key.pem
    - --etcd-servers=https://10.0.0.3:2379,https://10.0.0.4:2379,https://10.0.0.5:2379
    - --bind-address=0.0.0.0
    - --secure-port=6443
    - --tls-cert-file=/etc/kubernetes/pki/kubernetes.pem
    - --tls-private-key-file=/etc/kubernetes/pki/kubernetes-key.pem
    - --insecure-port=8080
    - --profiling
    - --anonymous-auth=false
    - --client-ca-file=/etc/kubernetes/pki/ca.pem
    - --enable-bootstrap-token-auth
    - --requestheader-allowed-names="aggregator"
    - --requestheader-client-ca-file=/etc/kubernetes/pki/ca.pem
    - --requestheader-extra-headers-prefix="X-Remote-Extra-"
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --service-account-key-file=/etc/kubernetes/pki/ca.pem
    - --authorization-mode=Node,RBAC
    - --runtime-config=api/all=true
    - --enable-admission-plugins=NodeRestriction
    - --allow-privileged=true
    - --apiserver-count=3
    - --event-ttl=336h
    - --kubelet-certificate-authority=/etc/kubernetes/pki/ca.pem
    - --kubelet-client-certificate=/etc/kubernetes/pki/kubelet-client.pem
    - --kubelet-client-key=/etc/kubernetes/pki/kubelet-client-key.pem
    - --kubelet-https=true
    - --kubelet-timeout=10s
    - --proxy-client-cert-file=/etc/kubernetes/pki/proxy-client.pem
    - --proxy-client-key-file=/etc/kubernetes/pki/proxy-client-key.pem
    - --service-cluster-ip-range=172.16.0.0/16
    - --service-node-port-range=30000-32767
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --logtostderr=true
    - --v=2
    image: tpaas-registry.jdcloud.com/k8s/kube-apiserver:v1.14.6-28.607c9a4
    imagePullPolicy: "IfNotPresent"
    livenessProbe:
      failureThreshold: 80
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 8080
        scheme: HTTP
      initialDelaySeconds: 150
      timeoutSeconds: 150
    name: kube-apiserver
    resources:
      requests:
        cpu: 250m
    volumeMounts:
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs

```

* 验证 该pod 不受 K8S api-server 控制

```shell
#在节点上确认api-server进程, 可以看到运行时长 
#可以通过ps 也可以通过docker

$ docker ps | grep kube-apiserver
03dfdf3e5d31        cfdc5b749e56                                     "kube-apiserver --de…"   5 days ago          Up 5 days                               k8s_kube-apiserver_kube-apiserver-zhangjf-m-3_kube-system_66cafd7c3818b049af1bdad20af6b906_4
c1ca12435e5e        tpaas-registry.jdcloud.com/k8s/pause-amd64:3.1   "/pause"                 5 days ago          Up 5 days                               k8s_POD_kube-apiserver-zhangjf-m-3_kube-system_66cafd7c3818b049af1bdad20af6b906_5

$ ps aux | grep -E "TIME|kube-apiserver"

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      9546  1.3 11.3 755456 441680 ?       Ssl  Sep01 107:51 kube-apiserver --default-not-ready-toleration-seconds=360  --default-unreachable-toleration-seconds=360  ...

#通过K8S API server查看pod
$ kubectl get pod -n kube-system |grep api
kube-apiserver-m-1            1/1     Running   4          6d12h

#通过api server 删除该pod
$  kubectl delete pod kube-apiserver-m-1 -n kube-system
pod "kube-apiserver-m-1" deleted

#通过命令行看（即K8S api server）运行时间变为刚启动
#因为AGE是根据 kubelet向api server注册 mirror Pod时间算起
$ kubectl get pod -n kube-system |grep -E "AGE|api"
NAME                                  READY   STATUS    RESTARTS   AGE
kube-apiserver-m-1            1/1     Running   4          10s

#再次通过ps查看实际进程运行时间，通过TIME可以判断该服务没有重启
$ ps aux | grep -E "TIME|kube-apiserver"

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      9546  1.3 11.5 755456 449676 ?       Ssl  Sep01 107:58 kube-apiserver --default-not-ready-toleration-seconds=360 --default-unreachable-toleration-seconds=360 ...

#docker 验证没有重启
$ docker ps | grep kube-apiserver
03dfdf3e5d31        cfdc5b749e56                                     "kube-apiserver --de…"   5 days ago          Up 5 days                               k8s_kube-apiserver_kube-apiserver-zhangjf-m-3_kube-system_66cafd7c3818b049af1bdad20af6b906_4
c1ca12435e5e        tpaas-registry.jdcloud.com/k8s/pause-amd64:3.1   "/pause"                 5 days ago          Up 5 days                               k8s_POD_kube-apiserver-zhangjf-m-3_kube-system_66cafd7c3818b049af1bdad20af6b906_5
[root@zhangjf-m-3 manifests]#

```

