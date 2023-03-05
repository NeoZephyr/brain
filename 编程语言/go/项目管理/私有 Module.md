## 本地 module

module a 依赖 module b，module b 还没有发布到公共托管站点上。可以借助 go.mod 的 replace 指示符，来解决这个问题

```
require github.com/user/b v1.0.0
replace github.com/user/b v1.0.0 => module b的本地源码路径
```

## GOPROXY 服务

![[Pasted image 20230219110749.png]]

在每个开发机上，配置公共 GOPROXY 服务拉取公共 Go Module，同时再把私有仓库配置到 GOPRIVATE 环境变量。这样，所有私有 module 的拉取，都会直连代码托管服务器，不会走 GOPROXY 代理服务，也不会去 GOSUMDB 服务器做 Go 包的 hash 值校验

![[Pasted image 20230219110807.png]]

更多的公司 / 组织，可能会将私有 Go Module 放在公司 / 组织内部的 vcs 服务器上

![[Pasted image 20230219110819.png]]

搭建一个内部 goproxy 服务。一来，为那些无法直接访问外网的开发机器提供拉取外部 Go Module 的途径，二来，由于 in-house goproxy 的 cache 的存在，还可以加速公共 Go Module 的拉取效率。另外，对于私有 Go Module，开发机只需要将它配置到 GOPRIVATE 环境变量中就可以了，这样 Go 命令在拉取私有 Go Module 时，就不会再走 GOPROXY，而会采用直接访问 vcs 的方式拉取私有 Go Module

![[Pasted image 20230219110836.png]]

将外部 Go Module 与私有 Go Module 都交给内部统一的 GOPROXY 服务去处理，开发者只需要把 GOPROXY 配置为 in-house goproxy，就可以统一拉取外部 Go Module 与私有 Go Module。但由于 go 命令默认会对所有通过 goproxy 拉取的 Go Module，进行 sum 校验，而私有 Go Module 在公共 sum 验证 server 中又没有数据记 录。因此，开发者需要将私有 Go Module 填到 GONOSUMDB 环境变量中，这样，go 命令就不会对其进行 sum 校验了

goproxy.io 在官方站点给出了企业内部部署的方法，可以基于 goproxy.io 来实现