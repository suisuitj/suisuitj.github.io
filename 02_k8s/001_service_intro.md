# service

> service代表一类（通过label过滤）pod集合，对部署应用
>
> 1. 为代理的后端pod集合提供统一入口（ip/域名+端口号）
> 2. 提供负载均衡作用

## service类型

| 类型           | 使用场景                                                     | 备注       |
| -------------- | ------------------------------------------------------------ | ---------- |
| `ClusterIP`    | 集群内访问使用                                               | 默认类型   |
| `NodePort`     | 集群外可访问，默认开放端口30000-32767                        |            |
| `Headless`     | 不分配ip，无lb等功能场景使用                                 |            |
| `LoadBalancer` | 使用外部LB负责负载均衡，常用与云厂商k8s集群集成ELB使用       | 依赖外部LB |
| `ExternalName` | 类似cname，将该service流量转发到指定的目的（eg.其它ns的service） |            |

## 示例

1. CLusterIP Service

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
   spec:
     selector:
       app: MyApp
     ports:
       - protocol: TCP
         port: 80
         targetPort: 9376
   ```

2. NodePort Service

   k8s会在每个节点上对nodePort service暴露的端口进行监听，即每个nodePort Service都会占用集群中节点实例的一个端口。一个集群默认最大支持创建2768个NodePort Service

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
   spec:
     type: NodePort
     selector:
       #通过label过滤后端pod，k8s会自动创建对应的endpoints
       app: MyApp
     ports:
         #port 为service端口，一般设置成targetPort，即与对应endpoints的POD端口一致
       - port: 80
         #对应pod端口
         targetPort: 80
         #可选参数，一般不写nodePort，系统自动生成(default: 30000-32767)；手动填写注意全局唯一，避免冲突
         nodePort: 30007
   ```

3. Headless service

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
   spec:
     selector:
       app: MyApp
     #不分配service ip
     clusterIP: None
     ports:
       - protocol: TCP
         port: 80
         targetPort: 9376
   ```

4. LoadBalancer service

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
   spec:
     selector:
       app: MyApp
     ports:
       - protocol: TCP
         port: 80
         targetPort: 9376
     type: LoadBalancer
   ```

   如果k8s集群没有对应外部LB，该service的externalIP状态为pending

5. ExternalName


   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
     namespace: prod
   spec:
     type: ExternalName
     externalName: my.database.example.com
   ```

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: my-service
   spec:
     #映射其它namespace的service
     externalName: my-another-service.another_namespace.svc.cluster.local
     ports:
     - port: 80
       protocol: TCP
       targetPort: 9376
     sessionAffinity: None
     type: ExternalName
   ```


6. 通过service引入集群外部服务

   service不使用selector时，k8s不会自动创建endpoints。手动创建`同名` endpoints, 在endpoints中指定外部服务地址。

   ```yaml
   kind: Service
   apiVersion: v1
   metadata:
     name: my-service
   spec:
     ports:
       - name: my-srv
         protocol: TCP
         port: 80
         targetPort: 9376
   ---
   kind: Endpoints
   apiVersion: v1
   metadata:
     name: my-service
   subsets: 
     - addresses:
       - ip: 192.168.0.1
       - ip: 192.168.0.2
       - ip: 192.168.0.3
       ports:
       - port: 9376
         name: my-back
   ```


## service实现机制

service IP 并不是真实分配的ip地址，只是在路由转发规则使用。

service通过k8s集群节点上的kube-proxy实现，支持3种代理模式： `User space`, `iptables`, `IPVS`。这里忽略User space。

1. iptables proxy mode

      ![iptable-proxy-mode](001_service_intro\services-iptables-overview.svg)

   * kube-proxy为每一个service创建/维护一条路由转发规则。通过路由转发规则将访问service ip和端口的流量转发至对应后端pod IP。

   * 不支持多样的负载均衡策略，比如lc等

   * 集群规模很大时，性能低下

   * 路由转发策略默认是`round-robin`, 可以通过配置`SessionAffinity`， 支持session保持

     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: my-service
     spec:
       selector:
         app: my-service
       sessionAffinity: ClientIP
       sessionAffinityConfig：
         clientIP：
           #会话保持时长
           timeoutSeconds：1800
       type: NodePort
       ports: 
       - port: 80
         targetPort: 80
         nodePort: 30080
     ```

2. IPVS proxy mode

   ![ipvs-proxy-mode](001_service_intro\services-ipvs-overview.svg)

   * kube-proxy调用`netlink`接口为每一个service创建/维护IPVS规则。IPVS负责流量转发

   * IPVS底层使用哈希表映射转发关系，并且工作在内核空间，性能更高

   * 支持多种路由转发策略 `rr` `lc` `dg` `sh` `sed` `nq`
