## 四. 部署dashboard
```
V2.0.0 相比 V1.x.x 优势
监控信息不需要通过 Heapster 来提供，而是通过 Metrics Server 来提供，Metrics Scraper服务来采集，不需要单独维护 Heapster
支持暗黑主题
监控图显示更细节化
编辑支持 yaml 和 json
v2.0.0-beta6 兼容性
Kubernetes版本	兼容性
1.12	?
1.13	?
1.14	?
1.15	?
1.16	✓
✓ 完全支持的版本范围
? 由于 Kubernetes API 版本之间的重大更改，某些功能可能无法在仪表板中正常使用。
```
### 1. 生成证书
下面是生成 k8s dashboard 域名证书方法，任何一种都可以
```
通过 https://freessl.cn 网站，在线生成免费1年的证书
通过 Let’s Encrypt 生成 90天 免费证书
通过 Cert-Manager 服务来生成和管理证书
```
v2.0.0 单独放一个 namespace，下面是创建 kubernetes-dashboard namespace
```
$ kubectl  create namespace kubernetes-dashboard
```
把生成的免费证书存放在 $HOME/certs 目录下，取名为 tls.crt 和 tls.key
```
$ mkdir $HOME/certs
```
创建 ssl 证书 secret
```
$ kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kubernetes-dashboard
```
### 2. 准备dashboard yaml
```
$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
$ scp recommended.yaml <user>@<node-ip>:/etc/kubernetes/addons/
```
修改 Deployment yaml 配置， 把创建 kubernetes-dashboard-certs Secret 注释掉，前面已通过命令创建，具体修改见下面配置
 ```
 $ vi /etc/kubernetes/addons/recommended.yaml
#apiVersion: v1
#kind: Secret
#metadata:
#  labels:
#    k8s-app: kubernetes-dashboard
#  name: kubernetes-dashboard-certs
#  namespace: kubernetes-dashboard
#type: Opaque
#添加ssl证书路径，关闭自动更新证书，添加多长时间登出

      containers:
      - args:
        #- --auto-generate-certificates
        - --tls-cert-file=/tls.crt
        - --tls-key-file=/tls.key
        - --token-ttl=3600
```
### 3. 部署 k8s dashboard 创建服务
```
$ kubectl apply -f /etc/kubernetes/addons/recommended.yaml
```
查看服务运行情况
```
$ kubectl get all -n kubernetes-dashboard
```
### 4. 创建登陆用户
```
$ cat /etc/kubernetes/addons/dashboard-admin.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kubernetes-dashboard
 
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kubernetes-dashboard
 ```
 创建
 ```
 $ kubectl apply -f /etc/kubernetes/addons/dashboard-admin.yaml
 ```
 ### 5. 创建Ingress 入口文件
 ```
$ cat /etc/kubernetes/addons/dashboard-ingress.yaml
 apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-das-ingress
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - das.xxx.ga
    secretName: das.xxx.ga
  rules:
  - host: das.xxx.ga
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
 ```
 准备ssl证书，注意Ingress必须与secret在同一个namespace
 ```
 $ kubectl -n kubernetes-dashboard create secret tls das.xxx.ga --key $HOME/certs/tls.key --cert $HOME/certs/tls.crt
```
创建访问入口
```
kubectl apply -f /etc/kubernetes/addons/dashboard-ingress.yaml
```
### 6. 访问dashboard
获取登陆 token
```
$ kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep kubernetes-dashboard-admin-token | awk '{print $1}')

https://das.xxx.ga:8443/
```
