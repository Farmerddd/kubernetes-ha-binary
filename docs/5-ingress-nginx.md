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
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./tls.key -out ./tls.crt  -subj "/CN=k8s.molbase.org/O=dev"
$ kubectl create secret tls k8s.molbase.org --key ./tls.key --cert ./tls.crt 

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
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  tls:
  - hosts:
    - k8s.molbase.org
    secretName: k8s.molbase.org
  rules:
  - host: k8s.molbase.org
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

