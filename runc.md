导出docker镜像并解压
docker export $(docker create alpine:3.12) | tar -C rootfs -xvf -
生成配置文件 rc spec
创建 rc run -d abc > app.out 2>&1
查看 rc list

rc run -d web1 > web1.out 2>&1
rc kill web1
rc delete web1

安装 bridge 网桥工具
yum install -y bridge-utils
brctl show # 查看网桥
brctl addbr jtthink0  #添加网桥 addbr ==> add bridge
brctl show
ip link set jtthink0 up # 启动网桥
ip addr add 10.12.0.1/24 dev jtthink0 # 设置网桥地址
brctl show
ip a
# 添加 peer 对
ip link add name veth0-host type veth peer name veth0-ns
# 启动veth0-host
ip link set veth0-host up
# 把 veth0-host 与网桥 jtthink0 关联起来
brctl addif jtthink0 veth0-host
ip a
# 添加网络命名空间 mycontainer
ip netns add mycontainer
# veth0-ns 加入 mycontainer
ip link set veth0-ns netns mycontainer
ip netns
ip netns list
# 查看命名空间 mycontainer ip信息
ip netns exec mycontainer ip a
# 把 veth0-ns 名称改为 eth0 ==> 问了逼格
ip netns exec mycontainer ip link set veth0-ns name eth0
# 设置 eth0 ip 地址
ip netns exec mycontainer ip addr add 10.12.0.2/24 dev eth0
# 启动 eth0
ip netns exec mycontainer ip link set eth0 up
# 设置回环地址
ip netns exec mycontainer ip addr add 127.0.0.1 dev lo
# 把回环启动
ip netns exec mycontainer ip link set lo up
# 把 ip 路由搭载在 10.12.0.1，也就是 jtthink0 上
ip netns exec mycontainer ip route add default via 10.12.0.1
# 查看ip信息
ip netns exec mycontainer ip a
# ping
ip netns exec mycontainer ping 127.0.0.1
ip netns exec mycontainer ping 10.12.0.1
ip netns exec mycontaner ping 10.12.0.2
ip netns exec mycontainer ping 10.12.0.2
ip netns list
rc list
# 查看网络命名空间
ls /var/run/netns
# rc 启动容器，名称为 abc
rc run -d abc > abc.out 2>&1
# rc 启动容器，名称为 web1
rc run -d web1 > web1.out 2>&1
rc list
# rc 启动容器，名称为 web2
rc run -d web2 > web2.out 2>71
rc list
# curl
ip netns exec mycontainer ping 10.12.0.2:8081
ip netns exec mycontainer curl 10.12.0.2:8081
ip netns exec mycontainer curl 10.12.0.2:8082

rc events web1
rc pause web1
ip netns exec mycontainer curl 10.12.0.2:80813
ip netns exec mycontainer curl 10.12.0.2:8081

创建pod共享网络
 rc list
# 查看进程ID为1546命名空间
ls /proc/1546/ns
# bridge 网桥的命名空间
ls /var/run/netns/
ip list
# 查看 bridge 网桥命名空间
ip netns
rc list
# 软链接把  /proc/1546/ns/net 指向 /var/run/netns/proc1546
# 只要放到这个路径，就会有对于命名空间生成
ln -s /proc/1546/ns/net /var/run/netns/proc1546
# 查看软链接是否生成
ls /var/run/netns/

# 创建 peer 对，pause 容器使用
ip link add name veth0-pause type veth peer name veth0-pause-ns
# 启动 veth0-pause
ip link set veth0-pause up
# 把 veth0-pause 加入到 jtthink0
brctl addif jtthink0 veth0-pause
# 设置 set veth0-pause-ns 命名空间为 proc1546 ==> 之前软链接生成的命名空间
ip link set veth0-pause-ns netns proc1546
# 把 eth0-pause-ns 改名为 eth0
ip netns exec proc1546 ip link set veth0-pause-ns name eth0
# 设置 eth0 ip 地址
ip netns exec proc1546 ip addr add 10.12.0.4/24 dev eth0
# 启动 eth0
ip netns exec proc1546 ip link set eth0 up
# 把 ip 路由只想 jtthink0 网桥
ip netns exec proc1546 ip route add default via 10.12.0.1
ip netns
# 查看命名空间 proc1546 ip 信息
ip netns exec proc1546 ip a
# ping 是否通行
ip netns exec proc1546 ping 127.0.0.1
ip netns exec proc1546 ping 10.12.0.1
ip netns exec proc1546 ping 10.12.0.4
ip netns exec proc1546 ping 10.12.0.2


config.json 文件
"namespaces": [
  {
          "type": "pid",
          "path": "/proc/1546/ns/pid"  #与 pause 容器（pid=1546）共享 pid
  },
  {
          "type": "network",
          "path": "/proc/1546/ns/net"  #与 pause 容器（pid=1546）共享 network
  },
  {
          "type": "ipc",
          "path": "/proc/1546/ns/ipc"  #与 pause 容器（pid=1546）共享 ipc
  }
]

# 执行命令，看到自己 pid 外，还可以看到其他进程的 pid
rc exec -t web1 ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    5 root      0:00 /app/main -p 8082
   11 root      0:00 /app/main -p 8081
   17 root      0:00 ps -ef


"linux": {
    "resources": {
          "cpu": {   # 现在cpu
                  "quota": 1000,
                  "period": 100000
          }
    }
}