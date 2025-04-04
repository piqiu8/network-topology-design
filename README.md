<!-- markdownlint-disable -->

<div align="center">
  <img alt="Image" src="https://github.com/piqiu8/network-topology-design/blob/master/desigin-image.png">
  <br>
  <br>
  <div>
    <img alt="simulator" src="https://img.shields.io/badge/simulator-ENSP-blue.svg">
    <img alt="protocol" src="https://img.shields.io/badge/protocol-IPv4%20%7C%20IPv6%20%7C%20BGP-blueviolet.svg">
  </div>
  <br>
</div>

<!-- markdownlint-restore -->

## 使用技术

`GVRP` `链路聚合` `DHCP` `DHCPv6` `MSTP` `AC旁挂` `VRRP` `VRRP6` `防火墙双机热备` `OSPF` `OSPFv3` `RIP` `RIPng` `ISIS` `ISISv6` `BGP` `BFD` `GRE over IPsec` `ACL` `路由策略` `NAPT` `NAT Server` `NAT64`

## 具体设计

### IP规划

#### IPv4地址

- 总部使用B类IPv4地址，分部使用C类IPv4地址，平台服务器使用A类IPv4地址，但因实际操作失误，公司总部与分部两者IPv4地址规划与计划相反，在此告知，IPv4地址规划请以实际设计为准
- 具体IPv4地址将采用如192.168.xy.x的形式，如x的IPv4地址为192.168.xy.x，y的IPv4地址为192.168.xy.y，x与y表示各自的设备序号，主机IPv4地址使用DHCP自动分配，无线的管理VLAN IPv4地址单独配置。公网IPv4地址则采用A类公网地址，具体形式与上述相同，环回口则配置为x.x.x.x/32形式，x为设备序号

#### IPv6地址

- 本设计中IPv6地址统一使用2001 : 0db8 :: /64
- 具体形式和IPv4具体形式相同，主机IPv6地址使用DHCPv6自动分配，ENSP中因无线不支持IPv6，所以无线部分不配置IPv6地址，公网部分为防止地址冲突，统一以2001 :: 0db8 : 112 :: /64为前缀，环回口则配置为x :: x/128形式，x为设备序号

### 公司总部设计

主要采用VRRP+MSTP的设计

- SW2、SW3、SW4为接入层设备，进行MSTP相关配置，所有实例中优先级都为8192，并在主机接入端口配置边缘端口优化网络
- LSW1和LSW2为核心层设备，配置VRRP与VRRP6确保链路冗余，并在两者间配置链路聚合，其次配置MSTP，分为实例1与实例2，VLAN 13、114、110走实例1，VLAN 17、16、24、100走实例2，实例1中LSW1作为根桥，实例2中LSW2作为根桥，并也在边缘部分配置边缘端口
- 接入层和核心层交换机配置GVRP动态学习VLAN，所有接口都设置为normal模式，DHCP服务器与AC的接口则设置为fixed模式
- DHCP服务器10配置DHCP与DHCPv6为总部财务部与总部技术部自动分配IPv4与IPv6地址。因需跨网段，需要DHCP RELAY，DHCP通过配置地址池与让LSW1作为中继来实现，DHCPv6采用有状态配置
- 对于无线部分，采用AC旁挂BRAS组网方式，DHCP服务器7为访客区提供IPv4地址，同样也配置地址池与中继，AC通过MAC认证方式绑定AP确保安全性，通过组方式配置多业务
- 对于出口层，使用FW8与FW9进行双机热备增强安全性，FW8为主，FW9为备，因防火墙上的管理策略不是本设计的重点，为方便操作，在安全策略中将默认动作全部改为允许。此外配置GRE over IPsec隧道，只有总部财务部、技术部与分部技术部可通过隧道进行互访
- 公司总部整体通过OSPF与OSPFv3进行连通，并配置边缘端口优化网络
- 配置BFD与OSPF、VRRP、VRRP6进行联动加快网络收敛，因ENSP中不支持BFD与OSPFv3进行联动，所以没有配置相应BFD
- 服务器机房只允许技术部进行访问，配置相应ACL进行访问控制，但因ENSP原因，交换机上关于IPv6的ACL无法生效，因此关于IPv6的访问控制只在路由器上进行示范配置
- 防火墙上设置NAPT进行IPv4地址转换，而IPv6本身虽可用NAT66进行转换，但目前ENSP并不支持该功能，所以改用NAT64将IPv6转换为IPv4进行通信，其中只有总部财务部、技术部与分部技术部可以访问外网服务

### 公司分部设计

- 公司分部设计总体与总部相似，但考虑到规模问题，取消使用VRRP，在关键部分改用链路聚合，并让LSW4作为MSTP的根桥
- 公司分部采用RIP与RIPng来进行连通
- 配置BFD与RIP进行联动，同样因ENSP中RIPng不支持配置BFD，所以没有配置相应BFD

### 平台服务器设计

- 同样为安全考虑，在路由器上配置NAT Server将私网服务器地址转换为公网地址再向公网发布

### 公网设计

整个公网都是基于BGP

- 将公网分为4个AS区域，分别为AS100、200、300、400
- 同一AS区域内使用ISIS与ISISv6进行连通，统一使用level-2
- 不同AS区域间使用BGP进行路由传递，EBGP之间使用物理端口建立邻居关系，IBGP之间使用环回口建立邻居关系，将AR4与AR5配置为路由反射器来模拟公网设备众多的情况
- 配置BFD与ISIS、BGP进行联动
- 在AR1与AR6上配置路由策略来进行流量控制，使隧道流量走下方链路，通过NAT转换的访问流量走上方链路

## 存在问题

~~(多半又是ENSP的BUG)~~

- VRRP6ping不通虚拟网关，无法正常生效 (我试出了个[邪道方法](https://blog.csdn.net/m0_67832087/article/details/136608267)，可以ping通虚拟网关，但意义不大)
- AC的BFD无法生效，OSPFv3的BFD只能在防火墙上开
- IPv6在断开主防火墙并在短时间内恢复主防火墙时可正常通信，但若在长时间不恢复主防火墙的情况下恢复主防火墙则会导致IPv6通信完全断开，只有重新运行才可恢复
- 如有其他问题，请先看issue中是否提及，若无再进行提问