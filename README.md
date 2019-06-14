#### 一、准备环境
```bash
$ cd /home/tools
# 安装 gcc 编译环境
$ yum install gcc
# 下载源码包，备用下载地址：https://github.com/firechiang/haproxy-test/raw/master/data/haproxy-1.9.8.tar.gz
$ wget https://www.haproxy.org/download/1.9/src/haproxy-1.9.8.tar.gz
```

#### 二、编译安装（注意：编译好的安装包可以直接发到其它机器运行）
```bash
$ cd /home/tools
# 解压到当前目录
$ tar -zxvf haproxy-1.9.8.tar.gz -C ./                 
$ cd haproxy-1.9.8
# 编译安装到 /usr/local/haproxy 目录，最后将测试的配置文件拷贝到安装目录（TARGET=linux31 是指编译成 Linux 3.1 内核使用）
$ sudo make TARGET=linux31 ARCH=x86_64 && sudo make install PREFIX=/usr/local/haproxy && sudo scp -r ./tests /usr/local/haproxy
```

#### 三、创建配置文件[vi /usr/local/haproxy/haproxy.cfg]
```bash
#logging options
global
    log 127.0.0.1 local0 info
    # 每个haproxy进程所接受的最大并发连接数
    maxconn 5120
    chroot /usr/local/haproxy
    uid 99
    gid 99
    daemon
    quiet
    nbproc 20
    # 进程ID文件所在目录
    pidfile /usr/local/haproxy/haproxy.pid
defaults
    log global
    # 使用4层代理模式，”mode http”为7层代理模式
    mode tcp
    # if you set mode to tcp,then you nust change tcplog into httplog
    option tcplog
    option dontlognull
    retries 3
    option redispatch
    # 每个haproxy进程所接受的最大并发连接数
    maxconn 2000
    contimeout 5s
    # 客户端空闲超时时间为 60秒 则HA 发起重连机制
    clitimeout 60s
    # 服务器端链接超时时间为 15秒 则HA 发起重连机制
    srvtimeout 15s	
    # front-end IP for consumers and producters
listen rabbitmq_cluster
    # 绑定访问主机和端口（注意：端口最好不要和代理服务使用相同的端口以免发生冲突）
    bind server003:5673
    # 配置TCP模式
    mode tcp
    #balance url_paramuserid
    #balance url_paramsession_idcheck_post 64
    #balance hdr(User-Agent)
    #balance hdr(host)
    #balance hdr(Host) use_domain_only
    #balance rdp-cookie
    #balance leastconn
    #balance source //ip
    # 简单的轮询
    balance roundrobin
    # 代理rabbitmq集群节点配置 #inter 每隔五秒对mq集群做健康检查， 2次正确证明服务器可用，2次失败证明服务器不可用，并且配置主备机制
    server server001 server001:5672 check inter 5000 rise 2 fall 2
    server server002 server002:5672 check inter 5000 rise 2 fall 2
    server server003 server003:5672 check inter 5000 rise 2 fall 2
#配置haproxy web监控，查看统计信息
listen stats
    # 开启 web 监控
    stats enable
    bind server003:8100
    mode http
    option httplog
    # 设置haproxy监控地址为http://localhost:8100/rabbitmq-stats
    stats uri /rabbitmq-stats
    # 监控台 5s 刷新一次
    stats refresh 5s
    # 密码框提示文本
    stats realm haproxy_admin
    # 认证用户名和密码
    stats auth jiang:jiang
    # 管理界面，成功登陆后可通过webui管理节点
    stats admin if TRUE
    # 隐藏HAProxy的版本号
    #stats hide-version
```

#### 四、分发安装包到其它机器
```bash
$ scr -r /usr/local/haproxy root@server004:/usr/local/haproxy
```

#### 五、启动和停止HAProxy（我们配置的监控访问地址：http://server06:8100/rabbitmq-stats）
```bash
$ cd /usr/local/haproxy/sbin
$ sudo ./haproxy -f /usr/local/haproxy/haproxy.cfg      # 启动 HAProxy，-f 是指定配置文件
$ sudo ps -ef | grep haproxy                            # 查看 HAProxy 进程状态
$ sudo kill `cat /usr/local/haproxy/haproxy.pid`        # 停止 HAProxy
```

#### 六、Web管理端控制后端节点上下线，用得比较多的2个选项是READY（就绪状态）和MAINT（维护状态）（注意：需要在配置文件启动该功能：listen stats > stats admin if TRUE）
```bash
1，MAINT 表示被勾选的节点需要进行维护，Apply进入维护状态后，Haproxy将会停止往这些节点转发请求，并等待已有的请求结束连接
2，READY 表示被勾选的节点已经完成维护，Apply进入就绪状态后，Haproxy会自动发起健康检查，如果检查通过，这些节点将进入映射状态，接受映射请求了。
```
![image](https://github.com/firechiang/haproxy-test/blob/master/image/manager-simple-use.png)


