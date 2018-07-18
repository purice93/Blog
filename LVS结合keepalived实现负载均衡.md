---
title: LVS结合keepalived实现负载均衡
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

>LVS可以实现负载均衡，但是不能够进行健康检查，比如一个rs出现故障，LVS 仍然会把请求转发给故障的rs服务器，这样就会导致请求的无效性。keepalived 软件可以进行健康检查，而且能同时实现 LVS 的高可用性，解决 LVS 单点故障的问题。

* 第一步-克隆主机
	* 直接进入虚拟机，删除临时网络配置，并配置网卡，修改后，`init 6`重启
	![][1]
	![修改ip和主机名][2]
	* xshell连接，主机为当前windows主机ip：192.168.116.X

* 安装配置keepalived，并需要保持主机时间同步

``` stylus
安装：Yum install  keepalived
配置：vi /etc/keepalived.conf
global_defs {
   notification_email {
     root@localhost#发送提醒邮件的目标地址可有多个
     goldbin@126.com
  }
   notification_email_from test@localhost#发送邮件的from地址，可以随意写，邮件地址不存在都无所谓
   smtp_server 127.0.0.1#邮件服务的地址，一般写本地
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state BACKUP# MASTER 主 和 从 
    interface eth0#VIP需要绑定的网卡名称
    virtual_router_id 51
    priority 101#优先级 主的优先级要高
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.116.200/24 dev eth0 label eth0:1 #设置VIP
    }
}

virtual_server 192.168.1.200 80 {#设置虚拟lvs服务，VIP PORT
    delay_loop 6
    lb_algo rr#调度算法wrr
    lb_kind DR#lvs的模式
    nat_mask 255.255.255.0
    persistence_timeout 50 同一个IP地址在50秒内lvs转发给同一个后端服务器
    protocol TCP

    real_server 192.168.116.3 80 {#设置真实服务器的心跳机制 RID PORT
        weight 1#权重
        HTTP_GET {#心跳检测的方式
            url {
              path /#心跳检查的地址
              status_code 200#心跳检查返回的状态
            }
            connect_timeout 2 #超时时间
            nb_get_retry 3#重复检查3次
            delay_before_retry 1#每隔1秒钟再次检查
        }
    }
    real_server 192.168.116.4 80 {#第二个真实服务器设置
        weight 1
        HTTP_GET {
            url {
              path /
              status_code 200
            }
            connect_timeout 2 
            nb_get_retry 3
            delay_before_retry 1
        }
    }
}

```


**失败，无法访问VIP，192.168.116.200，可以单独访问**
	


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521603644365.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1521603961594.jpg
