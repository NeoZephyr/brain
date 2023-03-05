```
global_defs {
  # 全局唯一
	router_id LVS
}

vrrp_instance VI {
	state MASTER
  
  # 绑定的网卡
  interface ens333
  
  # 保证主被节点一致
  virtual_router_id 51
  
  # 权重，一般 master 高于 backup
  priority 100
  
  # 主备之间同步检查时间间隔
  advert_int 2

  authentication {
  	auth_type PASS
    auth_pass 1111
  }
  
  # 虚拟出来的 ip，可以有多个
  virtual_ipaddress {
  	192.168.1.150
  }
}

# 配置集群访问地址
virtual_server 192.168.1.150 80 {
	# 健康检查时间
	delay_loop 6
  
  # 负载均衡算法
  lb_algo rr
  
  # LVS 模式
  lb_lind DR
  
  # 会话持久化时间
  persistence_timeout 10
  protocol TCP
  
  real_server 192.168.1.171 80 {
  	weight 1
    TCP_CHECK {
    	# 检查端口
    	connect_port 80
      
      # 超时时间
      connect_timeout 2
      
      # 重试次数
      nb_get_retry 2
      
      # 间隔时间
      delay_before_retry 3
    }
  }
  
  real_server 192.168.1.172 80 {
  	weight 1
    TCP_CHECK {
    	# 检查端口
    	connect_port 80
      
      # 超时时间
      connect_timeout 2
      
      # 重试次数
      nb_get_retry 2
      
      # 间隔时间
      delay_before_retry 3
    }
  }
}
```

清空规则

```bash
ipvsadm -C
```

重启 keepalived