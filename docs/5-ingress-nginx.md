# 五. 部署ingress-nginx
## 1. 部署ngress-nginx
```bash
# 下载ingress-nginx yaml
$ wget https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/mandatory.yaml
$ wget https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/provider/baremetal/service-nodeport.yaml

修改：service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
      nodePort: 8480
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
      nodePort: 8443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---


$ kubectl apply -f mandatory.yaml
$ kubectl apply -f service-nodeport.yaml
2. 生成证书
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./tls.key -out ./tls.crt  -subj "/CN=k8s.xxxx.org/O=dev"
$ kubectl create secret tls k8s.xxxx.org --key ./tls.key --cert ./tls.crt 

3. 
# 创建服务
$ cat nginx-ds.yml 

apiVersion: v1
kind: Service
metadata:
  name: svc-nginx-ds
  labels:
    app: svc-nginx-ds
spec:
  selector:
    app: pod-nginx-ds
  ports:
   - port: 80
     targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-nginx-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: pod-nginx-ds
  template:
    metadata:
      labels:
        app: pod-nginx-ds
    spec:
      containers:
      - name: rq-my-nginx
        image: nginx:1.12.0
        ports:
        - containerPort: 80

$ kubectl apply -f nginx-ds.yml

4. 创建nginx入口文件
cat nginx-ingress-ssl.yaml 

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ds-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false" #取消强制跳转到https
spec:
  tls:
  - hosts:
    - k8s.xxx.org
    secretName: k8s.xxx.org
  rules:
  - host: k8s.xxx.org
    http:
      paths:
      - path: /
        backend:
          serviceName: svc-nginx-ds
          servicePort: 80

kubectl apply -f nginx-ingress-ssl.yaml

5. 测试

http://192.168.15.246:8480
http://192.168.15.246:8443

6. 注意事项

将Nginx迁移到Ingress之后，通过日志系统发现日志里出现了很多“308”的状态码。

我们之前是http > http , https > https 这种模式，Ingress 开启TLS后，则是http>https , https>https 。所以是有是有差异的。

原因：默认情况下，如果为该Ingress启用了TLS，则控制器会使用308永久重定向响应将HTTP客户端重定向到HTTPS端口443。

k8s路由默认http跳转到https， 用的是308跳转，ie浏览器，或者有些低版本的浏览器不支持“308”跳转的，要改成“301”跳转，不然低版本的浏览器会报错。

介于以上问题，需要修改NGINX config map文件的以下配置：

1.ssl-redirect: "false" （服务端http强制调准到https，false 表示关闭，true表示打开）

2.hsts=false （客户端如（浏览器）强制跳转https,false表示关闭，ture表示打开）

如果需要开启强制跳转，那就使用301强制跳转，不要使用308

http-redirect-code = 301 （使用301 进行强制跳转）
