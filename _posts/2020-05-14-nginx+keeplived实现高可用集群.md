---
layout: post
title: 'nginx+keepalived部署'
date: 2020-05-14
author: blossom
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: 高可用 负载均衡 nginx keepalived
---

> nginx+keepalived服务集群部署

### 概述
一个服务的高可用是我们衡量服务稳定性的重要指标，往往我们的可用性指标都是在99.99%，那么如何保证服务的高可用呢？这不单单在业务代码层面需要关注，环境层面也是一个重要因素，
我们没办法保证服务是永不宕机的，所以我们需要搭建一个高可用的服务环境。
近期业务场景需要提供一个高可用+负载均衡的数据解析服务，保证agent上报的数据完整性，计划使用keepalived+nginx来实现高可用和请求负载均衡，保证一台服务故障，能自动切换
到备机服务。
### nginx
一款面向性能设计的http服务器，能反向代理http，https，smtp等协议请求，并且能够根据规则分发请求，达到负载均衡的目的。
### keepalived
基于VRRP协议实现的服务高可用方案，可以用来避免IP单点故障.
VRRP全称 Virtual Router Redundancy Protocol，即 虚拟路由冗余协议。 可以认为它是实现路由器高可用的容错协议，即将N台提供相同功能的路由器组成一个路由器组(Router Group)，这个组里面有一个master和多个backup，但在外界看来就像一台一样，构成虚拟路由器，拥有一个虚拟IP（vip，也就是路由器所 在局域网内其他机器的默认路由），占有这个IP的master实际负责ARP相应和转发IP数据包，组中的其它路由器作为备份的角色处于待命状态。 master会发组播消息，当backup在超时时间内收不到vrrp包时就认为master宕掉了，这时就需要根据VRRP的优先级来选举一个 backup当master，保证路由器的高可用。
ps：保证rs机器和vip同网段，目前没有研究机器异地keepalived服务怎么搭建，理论上应该要借助其他工具实现异地服务的高可用。
### 实际运用
#### 业务架构
![ngiinx](https://blossom102er.github.io/assets/img/nginx+keepalived.png)
暴露给外部服务的是VIP提供udp的端口，请求到VIP，由nginx提供请求转发，分发到实际的业务进程。其中，keepalived用来保证vip的高可用，当台转发的keepalived故障时，
备机可以主动接管服务，此处需要注意，假如nginx故障，keepalived正常时，服务是没有办法切换到备机的。因为keepalived的服务正常运行，备机没办法接管服务
所以，我们需要在keepalived配置增加后端服务的状态，如果发现后端服务异常，可以主动把keepalived进程干掉，保证vip的漂移。<br>
另外：需要提供UDP的反向代理，所以对nginx版本有要求，nginx1.9之后的版本需要在编译时指定--with-stream激活ngx_stream_core_module模块，此模块提供tcp以及udp的代理和负载均衡。
#### nginx安装
通过yum安装，首先根据linux的版本指定yum的repo源，安装步骤如下:

    #修改yum源
    [root@localhost ~]# vim /etc/yum.repos.d/nginx.repo
    [nginx]
    name=nginx repo
    baseurl=http://nginx.org/packages/centos/6/$basearch/
    gpgcheck=0
    enabled=1
    
    #这边安装nginx最新版本，直接会开启ngx_stream_core_module模块
    [root@localhost ~]# yum install nginx

    #修改配置文件，新增一下配置
    #注意 stream是http服务同级的 千万不要配置到http里面去了。
    [root@localhost ~]# vim /etc/nginx/nginx.conf
    stream {
       upstream goflow {
            hash $remote_addr consistent; #一致hash分配
            server 127.0.0.1:2312;
            server 127.0.0.2:2312;
       }
     
       server{
         listen 1231 udp; #对外暴露的服务端口
         proxy_pass goflow;
      }
    }
 #### keepalived安装
 通过源码安装keepalived，安装步骤如下：（主从备机一样搭建，配置略有不同，可以参照注释）
 
    #下载keeplived并安装
    [root@localhost src]# wget https://www.keepalived.org/software/keepalived-2.0.20.tar.gz
    [root@localhost src]# tar xvf  keepalived-2.0.20.tar.gz
    [root@localhost src]# cd  keepalived-2.0.20
    [root@localhost src]# ./configure --prefix=/usr/local/keepalived
    [root@localhost src]# make && make install
     
    #配置
    #主从配置差不多，只需要改由注释的那几行
    [root@localhost src]# vim /etc/keepalived/keepalived.conf
    ! Configuration File for keepalived
    global_defs {
       router_id nginx_1 #路由id 主备需要唯一，不能一样
       vrrp_skip_check_adv_addr
       vrrp_strict
       vrrp_garp_interval 0
       vrrp_gna_interval 0
    }
     
    vrrp_script chk_nginx {
        script "/etc/keepalived/nginx_check.sh" #检测后端服务可用性，如果服务不行，则权重会-20 所有 备机的权重应该是大于80 才能保证服务漂移
        interval 2
        weight 20
    }
     
    vrrp_instance VI_1 {
        state MASTER         #状态 主=MASTER  备机=BACKUP
        interface eth0       #与备机通讯的网卡
        virtual_router_id 134 #路由编号，主备必须一致
        priority 100         #优先级 备机必须小于主机的值
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        virtual_ipaddress {
            1.1.1.1       #vip 主备一样
    }
     
    #启动服务
    [root@localhost src]# /sbin/keepalived
    #查看是否挂了vip 1.1.1.1，和指定的interface网卡挂在一起
    [root@localhost src]# ip addr
    #查看备机的情况 并没有vip
    [root@localhost2 ~]# ip addr
    #干掉主机的keepalived 发现vip会漂移到备机，测试截图此处忽略

### 总结
以上完成nginx+keepalived高可用+负载均衡服务的搭建。
本次服务中采用的是nginx的stream模块，反向代理udp请求，本质上和http反向代理没有差别。
在ngx_stream_upstream_module这个模块上没有http的ip_hash功能，只能通过hash $remote_addr consistent设置。
具体配置可以参考官方docs说明：
http://nginx.org/en/docs/stream/ngx_stream_upstream_module.html
    


