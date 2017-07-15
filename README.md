# （2017/7）用OVS（模拟网关实现不同子网的互通）实现三层转发（同时用锐捷的S5750C交换机和RG-ONC控制器进行同样的实验）

1. 

用mininet和opendaylight建立如下拓扑：

h1 h1-eth0:s1-eth1

h2 h2-eth0:s1-eth2

s1 lo:  s1-eth1:h1-eth0 s1-eth2:h2-eth0

c0

然后，删除s1的所有流表。


2. 

将h1的ip设为10.0.0.1/8，h2的ip设为20.0.0.1/8。

给h2配置默认路由10.0.0.254，给h2配置默认路由20.0.0.254。

3.

给s1-eth1配置ip地址10.0.0.254/8，给s1-eth2配置ip地址20.0.0.254/8。

***

做了上面之些之后，如果是锐捷的S5750C交换机在没有开启openflow功能的情况下，h1，h2，10.0.0.254，20.0.0.254这4个地址已经可以ping通了。

S5750C在开启openflow功能之后，以及的OVS上，两台主机和交换机是ping不通的。

***

4.

首先的问题是，h1和h2与各自的网关是ping不通的，即h1和10.0.0.254是ping不通的。

通过tcpdump -vvv -i h1-eth0，tcpdump -vvv -i h2-eth0，tcpdump -vvv -i s1-eth1，tcpdump -vvv -i s1-eth2命令以及查看各设备arp表的方式，判断得出，问题是交换机收到arp报文之后并没有进行处理，所以交换机获取不到h1和h2的mac地址，所以h1和10.0.0.254 ping不通。

感觉是需要下发一些流表才能让交换机在收到arp报文之后进行处理，进而通过arp获取到h1的mac地址。

但是我们不知道应该下发什么流表实现这个效果。所以这里遇到了很难解决的问题。

5.

后来我又想到，在用s1 ping h1的时候，用tcpdump -vvv -i h1-eth0查看，发现交换机ping h1的时候，h1可以收到交换机发来的arp报文，并且会对该报文做出回复（s1-eth1也能收到该回复的报文，只是交换机没有做处理），这时用arp -a查看h1的arp表，发现h1已经暂时获取到了10.0.0.254的mac地址，在h1的arp表里存在10.0.0.254的mac地址时，用h1 ping 10.0.0.254,发出的就不再是arp报文，而是icmp报文（s1-eth1也能收到该icmp报文）。

于是想，即使交换机不能获取到h1和h2的mac地址，只要h1和h2能获取到10.0.0.254和20.0.0.254的mac，并在交换机上下发流表将ping的报文（icmp或ip？）进行转发，同时替换掉ping的报文中的目的mac地址，也许就可以使h1 ping通h2。

***

过程大概：

h1 ping h2时，由于h1和h2的ip地址不在同一网段，所以h1先用arp请求网关的mac，然后发icmp报文，源ip为h1，目的ip为h2,源mac为h1,目的mac为h1的网关。s1再发arp请求h2的mac，然后将从h1收到的icmp报文，源mac改为20.0.0.254的mac，目的mac改为h2。

***

所以在交换机上下发两条流表，使交换机对ping的报文进行转发，

ovs-ofctl add-flow s1 "table=0,priority=500,ip,nw_dst=10.0.0.1/32,actions=mod_dl_dst=0a:9c:4b:db:35:27,output:1"

ovs-ofctl add-flow s1 "table=0,priority=500,ip,nw_dst=20.0.0.1/32,actions=mod_dl_dst=ca:03:c6:01:06:0b,output:2"

（报文类型写ip或是icmp都能成功，至于为什么，还不懂

ovs-ofctl add-flow s1 "table=0,priority=500,icmp,nw_dst=10.0.0.1/32,actions=mod_dl_dst=0a:9c:4b:db:35:27,output:1"

ovs-ofctl add-flow s1 "table=0,priority=500,icmp,nw_dst=20.0.0.1/32,actions=mod_dl_dst=ca:03:c6:01:06:0b,output:2"

）

之后要做的就是让h1获得10.0.0.254的mac地址，让h2获得20.0.0.254的mac地址。

最理想的方式还是用arp来获得，但因为我们还不知道怎么下发流表才能让交换机处理arp报文，所以这种方式暂时用不了。

我暂时用了2种方式让h1获得10.0.0.254的mac地址，让h2获得20.0.0.254的mac地址，

1. 不断地让s1 ping h1和h2,这样，s1会不断地用arp把10.0.0.254和20.0.0.254的mac地址分别发给h1和h2。

2. 用arp -s <ip地址> <mac地址>命令将10.0.0.254和20.0.0.254的mac地址配置到h1和h2的arp表里。

测试之后发现，这两种方式都可以让h1成功ping通h2。

***

参考：

[Open vSwith模拟网关实现不同子网的互通 ](http://www.sdnlab.com/14101.html)

[Linux arp命令](http://www.lx138.com/page.php?ID=VkZkd1drMVJQVDA9)

[二层交换、路由和三层交换](http://blog.sina.com.cn/s/blog_43c625f101012euf.html)
