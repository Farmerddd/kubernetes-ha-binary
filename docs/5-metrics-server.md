## 五. 部署Metrics-server
```
与kubernets兼容性
Metrics Server	Metrics API group/version	Supported Kubernetes version
0.3.x	metrics.k8s.io/v1beta1	1.8+
0.2.x	metrics.k8s.io/v1beta1	1.8+
0.1.x	metrics/v1alpha1	1.7
```
### 安装前准备
```
生成证书
$ mkdir kubernetes-ha-binary/target/pki/metrics-server
$ cd kubernetes-ha-binary/target/pki/metrics-server

$ cat metrics-server.json
{
  "CN": "metrics-server",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "seven"
    }
  ]
}
```
生产证书，秘钥
```
$ cfssl gencert -ca=../ca.pem \
  -ca-key=../ca-key.pem \
  -config=../ca-config.json \
  -profile=kubernetes metrics-server.json | cfssljson -bare metrics-server
```  
在集群中创建secret
```
$ kubectl -n kube-system create secret generic metrics-server-certs --from-file=metrics-server-key.pem --from-file=metrics-server.pem
```
下发到每个worker节点
```
$ scp metrics-server*.pem master1:/etc/kubernetes/pki/metrics-server/
$ scp metrics-server*.pem worker1:/etc/kubernetes/pki/metrics-server/
$ scp metrics-server*.pem worker2:/etc/kubernetes/pki/metrics-server/
```
metrics-server 是扩展的 APIServer，依赖于kube-aggregator，因为我们需要在 APIServer 中开启相关参数,
查看 APIServer 参数配置，确保你的 APIServer 启动参数中包含下的一些参数配置。
如果您未在 master 节点上运行 kube-proxy，则必须确保 kube-apiserver 启动参数中包含--enable-aggregator-routing=true
```
  --proxy-client-cert-file=/etc/kubernetes/pki/metrics-server/metrics-server.pem \
  --proxy-client-key-file=/etc/kubernetes/pki/metrics-server/metrics-server-key.pem \
  --requestheader-client-ca-file=/etc/kubernetes/pki/ca.pem \
  --requestheader-allowed-names=aggregator \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --enable-aggregator-routing=true
```
### 安装metrics-server
```
$ git clone https://github.com/kubernetes-sigs/metrics-server.git
$ cd metrics-server/deploy/1.8+

修改metrics-server-deployment.yaml

  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: metrics-server                                                                 
    namespace: kube-system
    labels:
      k8s-app: metrics-server
  spec:
    selector:
      matchLabels:
        k8s-app: metrics-server
    template:
      metadata:
        name: metrics-server
        labels:
          k8s-app: metrics-server
      spec:
        serviceAccountName: metrics-server                                               
        volumes:
        # mount in tmp so we can safely use from-scratch images and/or read-only containers
        - name: tmp-dir
          emptyDir: {}
        - name: metrics-server-certs                                                     
          secret:
            secretName: metrics-server-certs                                             
        containers:
        - name: metrics-server                                                           
          image: k8s.gcr.io/metrics-server-amd64:v0.3.6                                  
          #args:
          command:
            #- --cert-dir=/tmp                                                           
            - /metrics-server
            - --secure-port=4443
            #- --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
            - --kubelet-preferred-address-types=InternalIP
            - --kubelet-insecure-tls
            - --tls-cert-file=/certs/metrics-server.pem
            - --tls-private-key-file=/certs/metrics-server-key.pem
          ports:
          - name: main-port
            containerPort: 4443
            protocol: TCP
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
          imagePullPolicy: Always
          volumeMounts:
          - name: tmp-dir
            mountPath: /tmp
          - name: metrics-server-certs
            mountPath: /certs
        nodeSelector:
```
创建授权文件
```
$ cat metrics-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-server
rules:
- apiGroups:
    - metrics.k8s.io
  resources:
    - pods
    - nodes
    - namespaces
  verbs:
    - get
    - list
    - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: metrics-server
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: system:metrics-server
```
创建服务
```
kubectl apply -f 1.8+/
```
测试
```
kubectl top nodes
kubectl top pods
```
