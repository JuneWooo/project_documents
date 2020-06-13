# Keepalived高可用集群应用

```mysql
#keepalibed 安装
yum  install -y keepalived
#主的负载均衡器和从的负载均衡器都要安装
#最好把防火墙关闭 selinux关闭
service iptables stop
setenforce 0
#给配置文件做一个备份
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.back
#配置keepalived
vim /etc/keepalived/keepalived.conf

#keepalived配置
global_defs {
   #当负载均衡器故障了以后,通知谁
   notification_email {
        460286007@qq.com
   }
   #当负载均衡器故障了以后,以谁的邮箱发邮件
   notification_email_from this_my_email@126.com
   #smtp邮件服务器的地址
   smtp_server smtp.126.com
   smtp_connect_timeout 30
   #路由id
   router_id lb_01
}

#虚拟机路由冗余协议
vrrp_instance lb {
	#主服务器
    state MASTER
    #使用那一块网卡
    interface eth0
    #主机和备用机器的 虚拟路由id 必须相同
    virtual_router_id 125
    #优先级 数字越大越优先  备用机的数字要小于主机
    priority 150
    #宕机切换时间  秒
    advert_int 1
    #验证
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    #虚拟的ip地址
    virtual_ipaddress {
        10.11.52.222
    }
}
```

```mysql
#虚拟机路由冗余协议
vrrp_instance lb {
	#备用服务器
    state BACKUP
    #使用那一块网卡
    interface eth0
    #主机和备用机器的 虚拟路由id 必须相同
    virtual_router_id 125
    #优先级 数字越大越优先  备用机的数字要小于主机
    priority 100
    #宕机切换时间  秒
    advert_int 1
    #验证
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    #虚拟的ip地址
    virtual_ipaddress {
        10.11.52.222
    }
}
```

```mysql
#主机和 备用机配置好以后
ip addr #查看虚拟地址
#主机的虚拟机地址会有两个,一个是自己的Ip地址,您一个是虚拟地址
link/ether 00:0c:29:6c:af:3d brd ff:ff:ff:ff:ff:ff
    inet 10.11.52.60/24 brd 10.11.52.255 scope global eth0
    inet 10.11.52.222/32 scope global eth0
    inet6 fe80::20c:29ff:fe6c:af3d/64 scope link 
       valid_lft forever preferred_lft forever
#备用机的地址只有一个,只有自身的ip地址,如果主机挂了,备用机才会接管虚拟地址,代替主机执行工作
    link/ether 00:0c:29:6c:af:3d brd ff:ff:ff:ff:ff:ff
    inet 10.11.52.61/24 brd 10.11.52.255 scope global eth0
    inet6 fe80::20c:29ff:fe6c:af3d/64 scope link 
       valid_lft forever preferred_lft forever   
       
#keepalived 设置开机启动,保证开机以后自动接管任务

#keepalived 放入linux启动管理体系中
chkconfig --add keepalived
 
#查看全部服务在各运行级状态
chkconfig --list keepalived
 
#只要运行级别35启动，其他都关闭
chkconfig --levels 35 keepalived on
```



## 三个重要功能

1. 管理LVS负载均衡软件
2. 实现对LVS集群节点健康检查功能（healthcheck）
3. 作为系统网络服务的高可用功能（failover）


## Keepalived高可用故障切换转移原理

VRRP，全称Virtual Router Redundancy Protocol，中文名为虚拟路由冗余协议，VRRP的出现就是为了解决静态路由的单点故障问题，VRRP是通过一种竞选机制来将路由的任务交给某台VRRP路由器的。



如果你在面试时，要你解答Keepalived的工作原理，建议用自己的话回答如下内容，以下为对面试官的表述：

Keepalived高可用对之间是通过VRRP通信的，因此，我从VRRP开始给您讲起：

1. VRRP，全称Virtual Router Redundancy Protocol，中文名为虚拟路由冗余协议，VRRP的出现是为了解决静态路由的单点故障。
2. VRRP是通过一种竞选协议机制来将路由任务交给某台VRRP路由器的。
3. VRRP用IP多播的方式（默认多播地址（224.0.0.18））实现高可用对之间通信。
4. 工作时主节点发包，备节点接包，当备节点接收不到主节点发的数据包的时候，就启动接管程序接管主节点的资源。备节点可以有多个，通过优先级竞选，但一般Keepalived系统运维工作中都是一对。
5. VRRP使用了加密协议加密数据，但Keepalived官方目前还是推荐用明文的方式配置认证类型和密码。



介绍完了VRRP，接下来我再介绍一下Keepalived服务的工作原理：

是通过竞选机制来确定主备的，主的优先级高于备，因此，工作时主会优先获得所有的资源，备节点处于等待状态，当主挂了的时候，备节点就会接管主节点的资源，然后顶替主节点对外提供服务。

在Keepalived服务对之间，只有作为主的服务器会一直发送VRRP广播包，告诉备它还活着，此时备不会抢占主，当主不可用时，即备监听不到主发送的广播包时，就会启动相关服务接管资源，保证业务的连续性。接管速度最快可以小于1秒。



## 开始实战

## 环境准备

四台机器，两台lb两台web

修改lb01的配置文件如下

```nginx
！ Configuration File for keepalived
global_defs {    
  notification_email {    
    1410428714@qq.com    
  }    
  notification_email_from Alexandre.Cassen@firewall.loc
  smtp_server 127.0.0.1    
  smtp_connect_timeout 30    
  router_id lb01     #<==id为lb01，不同的keepalived.conf此ID要唯一
}
  vrrp_instance VI_1 {     #<==实例名字为VI_1，相同实例的备节点名字要和这个相同    
  	state MASTER     #<==状态为MASTER，备节点状态需要为BACKUP    
    interface eth0     #<==通信接口为eth0，此参数备节点设置和主节点相同    
    virtual_router_id 55     #<==实例ID为55，keepalived.conf里唯一    
    priority 150     #<==优先级为150，备节点的优先级必须比此数字低
    advert_int 1     #<==通信检查间隔时间1秒    
    authentication {        
      auth_type PASS     #<==PASS认证类型，此参数备节点设置和主节点相同        
      auth_pass 1111     #<==密码是1111，此参数备节点设置和主节点相同    
    }    
    virtual_ipaddress {        
      10.0.0.12/24 dev eth0 label eth0:1#<==虚拟IP，即VIP为10.0.0.12，子网掩码为24位，绑定接口为eth0，别名为eth0：1，此参数备节点设置和主节点相同    
    }
  }
```

执行ip addr

```
ip addr | grep 10.0.0.12	#看看是不是多了一个虚拟ip，这个ip为真实的外网ip
```



修改lb02的配置文件

```nginx
！ Configuration File for keepalived
global_defs {    
  notification_email {    
    1410428714@qq.com    
  }    
  notification_email_from Alexandre.Cassen@firewall.loc
  smtp_server 127.0.0.1    
  smtp_connect_timeout 30    
  router_id lb02     #<==id为lb01，不同的keepalived.conf此ID要唯一
}
  vrrp_instance VI_1 {     #<==实例名字为VI_1，相同实例的备节点名字要和这个相同    
  	state BACKUP     #<==状态为MASTER，备节点状态需要为BACKUP    
    interface eth0     #<==通信接口为eth0，此参数备节点设置和主节点相同    
    virtual_router_id 55     #<==实例ID为55，keepalived.conf里唯一    
    priority 100     #<==优先级为150，备节点的优先级必须比此数字低
    advert_int 1     #<==通信检查间隔时间1秒    
    authentication {        
      auth_type PASS     #<==PASS认证类型，此参数备节点设置和主节点相同        
      auth_pass 1111     #<==密码是1111，此参数备节点设置和主节点相同    
    }    
    virtual_ipaddress {        
      10.0.0.12/24 dev eth0 label eth0:1#<==虚拟IP，即VIP为10.0.0.12，子网掩码为24位，绑定接口为eth0，别名为eth0：1，此参数备节点设置和主节点相同    
    }
  }
```

执行ip addr

```
ip addr | grep 10.0.0.12	#看看是不是多了一个虚拟ip，这个ip为真实的外网ip
```

关掉lb_01，看看lb__02能否接管



## 裂脑问题

由于某些原因，导致两台高可用服务器对在指定时间内，无法检测到对方的心跳消息，各自取得资源及服务的所有权，而此时的两台高可用服务器对都还活着并在正常运行，这样就会导致同一个IP或服务在两端同时存在而发生冲突，最严重的是两台主机占用同一个IP地址，当用户写入数据时可能会分别写入到两端，这可能会导致服务器两端的数据不一致或造成数据丢失，这种情况就被称为裂脑。



**一般来说，裂脑的发生，有以下几种原因：**   

1. 高可用服务器对之间心跳线链路发生故障，导致无法正常通信。  


2. 心跳线坏了（包括断了，老化）。  


3. 网卡及相关驱动坏了，IP配置及冲突问题（网卡直连）。  
4. 心跳线间连接的设备故障（网卡及交换机）。   
5. 仲裁的机器出问题（采用仲裁的方案）。  
6. 高可用服务器上开启了iptables防火墙阻挡了心跳消息传输。 
7. 高可用服务器上心跳网卡地址等信息配置不正确，导致发送心跳失败。  
8. 其他服务配置不当等原因，如心跳方式不同，心跳广播冲突、软件Bug等。
9. Keepalived配置里同一VRRP实例如果vir-tual_router_id两端参数配置不一致，也会导致裂脑问题发生。



**裂脑解决方案**

1. 同时使用串行电缆和以太网电缆连接，同时用两条心跳线路，这样一条线路坏了，另一个还是好的，依然能传送心跳消息。   
2. 当检测到裂脑时强行关闭一个心跳节点（这个功能需特殊设备支持，如Stonith、fence）。相当于备节点接收不到心跳消息，通过单独的线路发送关机命令关闭主节点的电源。   
3. 做好对裂脑的监控报警（如邮件及手机短信等或值班），在问题发生时人为第一时间介入仲裁，降低损失。例如，百度的监控报警短信就有上行和下行的区别。报警信息发送到管理员手机上，管理员可以通过手机回复对应数字或简单的字符串操作返回给服务器，让服务器根据指令自动处理相应故障，这样解决故障的时间更短。

**keepalived裂脑解决方案**



作为互联网应用服务器的高可用，特别是前端Web负载均衡器的高可用，裂脑的问题对普通业务的影响是可以忍受的，如果是数据库或者存储的业务，一般出现裂脑问题就非常严重了。因此，可以通过增加冗余心跳线路来避免裂脑问题的发生，同时加强对系统的监控，以便裂脑发生时人为快速介入解决问题。  

1. 如果开启防火墙，一定要让心跳消息通过，一般通过允许IP段的形式解决。  
2. 可以拉一条以太网网线或者串口线作为主被节点心跳线路的冗余。 
3. 开发监测程序通过监控软件（例如Nagios）监测裂脑。



下面是生产场景检测裂脑故障的一些思路：



1）简单判断的思想：只要备节点出现VIP就报警，这个报警有两种情况，一是主机宕机了备机接管了；二是主机没宕，裂脑了。不管属于哪个情况，都进行报警，然后由人工查看判断及解决。

2）比较严谨的判断：备节点出现对应VIP，并且主节点及对应服务（如果能远程连接主节点看是否有VIP就更好了）还活着，就说明发生裂脑了。