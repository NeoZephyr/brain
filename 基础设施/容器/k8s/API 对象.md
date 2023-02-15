## 对象属性

### TypeMeta

Kubernetes 对象的最基本定义，它通过引入 GKV 模型定义了一个对象的类型

Group: Kubernetes 将对象依据功能范围归入不同的分组，比如把支撑最基本功能的对象归入 core 组，把与应用部署有关的对象归入 apps 组
Kind: 定义一个对象的基本类型。比如 Node、Pod、Deployment 等
Version: 版本

### MetaData

Label: 标签
Annotation: 属性扩展
Finalizer
ResourceVersion

### Spec

用户的期望状态，由创建对象的用户端来定义

### Status

对象的实际状态，由对应的控制器收集实际状态并更新

## Pod

### 环境变量

直接设置值
```yaml
apiVersion: v1
kind: Pod
metadata:
	name: hello
spec:
	containers:
  - image: nginx:1.15
    name: alpine
    env:
    - name: ENV
      value: test
```

读取 Pod Spec 的某些属性
```yaml
env:
- name: POD_NAME
valueFrom:
	fieldRef:
		apiVersion: v1
			fieldPath: metadata.name
```

从 ConfigMap 读取某个值
```yaml
env:
- name: POD_NAME
valueFrom:
	configMapKeyRef:
  	name: my-env
    key: POD_NAME
```

从 Secret 读取某个值
```yaml
env:
- name: USERNAME
valueFrom:
	secretKeyRef:
  	name: my-secret
    key: username
```

### 存储卷

存储卷定义包括两个部分：
volume: 定义 Pod 可以使用的存储卷来源
volumeMounts: 定义存储卷如何 Mount 到容器内部

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: app
spec:
	containers:
  - image: nginx:1.15
    name: nginx
    volumeMounts:
    - name: data
      mountPath: /data
    volumns:
    - name: data
      emptyDir: {}
```

### 资源限制
```yaml
apiVersion: v1
kind: Pod
metadata:
	name: app
spec:
	containers:
  - image: nginx:1.15
    name: nginx
    resources:
    	limit:
      	cpu: "500m"
        memory: "128Mi"
```

## ConfigMap

将非机密性的数据保存到键值对

## Secret

保存和传递密码、密钥、认证凭证这些敏感信息

## User & Service Account

用户账户为人提供账户标识，人的身份与服务的 Namespace 无关，所以用户账户是跨 Namespace 的服务账户为计算机进程和 Pod 提供账户标识，与特定的 Namespace 相关

## Service

service 是应用服务的抽象，通过 labels 为应用提供负载均衡和服务发现。匹配 labels 的 Pod IP 和端口列表组成 endpoints，由 kube-proxy 负责将服务 IP 负载均衡到这些 endpoints 上面

每个 Service 都会自动分配一个 cluster IP 和 DNS 名，其他容器可以通过该地址或者 DNS 来访问服务