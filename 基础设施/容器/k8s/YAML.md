```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: port
              protocol: TCP
          volumeMounts:
            - name: nginx-cm
              mountPath: "/etc/nginx/conf.d"
      volumes:
        - name: nginx-cm
          configMap:
            name: nginx-cm

# 修改 config-map 查看 pod
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-cm
data:
  pain.conf: |-
    server {
      listen       80;
      server_name  www.pain.com;
      location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
      }
      error_page   500 502 503 504  /50x.html;
    }


apiVersion: v1
kind: Service
metadata:
  name: nginx-inbound
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx 
```