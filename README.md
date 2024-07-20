## pve lxc 安装tailscale

### 流程：

1.  要在非特权容器中启动 Tailscale，可以在 LXC 的配置中启用对 /dev/tun 设备的访问。
例如，使用 Proxmox 7.0 作为 ID 为 112 的非特权 LXC 进行托管，关闭LXC的情况下，在pve宿主机中将以下行将添加到 /etc/pve/lxc/112.conf 中：
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file


2.  安装 tailscale
2.1.1.    使用alpine安装，如果默认源安装的不是最新版，可以使用独立社区edge源
echo "@edge_community http://dl-cdn.alpinelinux.org/alpine/edge/community" | tee /etc/apk/repositories
apk add tailscale@edge_community

2.1.2.  使用全局社区edge源：
在 /etc/apk/repositories 中修改为： 
        http://dl-cdn.alpinelinux.org/alpine/edge/community
apk add tailscale

2.1.3.  或者在修改社区源后，使用官方脚本安装：
    curl -fsSL https://tailscale.com/install.sh | sh

2.1.4.  然后先把 tailscale 添加到开机自启，然后启动 tailscale 服务：
rc-update add tailscale
rc-service tailscale start

2.2.或者使用ubuntu24安装，添加源：
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list


2.3.  启动LXC，在LXC中运行以下代码，完成设备注册：
tailscale up


3.  启用IP转发，默认进行snat
3.1.如果您的 Linux 系统有/etc/sysctl.d目录，请使用：
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf

如果使用的是alpine，则：
echo 'net.ipv4.ip_forward=1' | tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding=2' | tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.eth0.accept_ra=2' | tee -a /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf

3.1.1.否则，使用：
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.conf
sysctl -p /etc/sysctl.conf

3.2.如果您的 Linux 节点使用firewalld，由于 已知问题，您可能还需要允许伪装。作为解决方法，您可以使用以下命令允许伪装：
firewall-cmd --permanent --add-masquerade

3.3.启用 IP 转发时，请确保您的防火墙设置为默认拒绝流量转发。这是常见防火墙（例如ufw 和 ）的默认设置firewalld，
    可确保您的设备不会路由您不希望的流量。当使用alpine时，需要启用 sysctl 自启动，仅限alpine：
rc-update add sysctl   
rc-service sysctl start

3.4.重新启动LXC

4.  阻止 Tailscaled 被覆盖/etc/resolv.conf，退出节点使用以下命令后自身dns不再受 MagicDNS 影响，将使用系统自身dns：
tailscale set --accept-dns=false



5.  将设备通告为子网路由器，默认添加参数 --accept-routes 以启用路由流入：
tailscale up --advertise-routes=192.168.50.0/24,192.168.100.0/24,192.168.150.0/24,10.0.0.0/24 --accept-routes



6.  将设备通告为退出节点 
    从您想要用作退出节点的设备，tailscale up使用该--advertise-exit-node标志以及您通常使用的任何其他标志重新运行：
tailscale up --advertise-exit-node

7.1.  tailscale 自动更新，使用以下代码依赖于tailscale自身更新，或者使用apk、yum、apt等命令更新：
tailscale set --auto-update

7.2.  tailscale 开启web设置页面：
tailscale set --webclient

7.3.  阻止其它tailnet设备传入连接，您的设备仍然可见并允许发送流量，但不会接受通过 Tailscale 的任何连接，包括 ping：
tailscale up --shields-up

8. 步骤5、6、7合一，可以使用 tailscale up --reset进行重置：
tailscale set --accept-routes --hostname=tailscale-us --accept-dns=false --webclient --auto-update

9. 使用tailscale网络登录 ip:5252，进行设置：
tailscale set --webclient




### 注意：
1.  使用的命令标志tailscale up是：
--advertise-routes：将物理子网路由公开到整个 Tailscale 网络。
--snat-subnet-routes=false：禁用源 NAT。在正常操作中，子网设备将看到源自子网路由器的流量。这简化了路由，但不允许穿越多个网络。通过禁用源 NAT，终端计算机将原始计算机的 LAN IP 地址视为源。
--accept-routes=true：接受其他子网路由器以及作为子网路由器的任何其他节点的通告路由。

2.  可选。Tailscale 版本 1.54 或更高版本与 Linux 6.2 或更高版本内核一起使用，可通过传输层卸载：
2.1.请确保以下网络设备配置到位以获得最佳结果：
NETDEV=$(ip route show 0/0 | cut -f5 -d' ')
ethtool -K $NETDEV rx-udp-gro-forwarding on rx-gro-list off

2.2.复制并运行以下命令来创建一个脚本，该脚本将在每次启动时配置这些设置：
printf '#!/bin/sh\n\nethtool -K %s rx-udp-gro-forwarding on rx-gro-list off \n' "$(ip route show 0/0 | cut -f5 -d" ")" | tee /etc/networkd-dispatcher/routable.d/50-tailscale
chmod 755 /etc/networkd-dispatcher/routable.d/50-tailscale

2.3.测试创建的脚本以确保它在您的计算机上成功运行：
/etc/networkd-dispatcher/routable.d/50-tailscale
test $? -eq 0 || echo 'An error occurred.'

3.现在alpine安装tailscale依赖于ip6tables，alpine3.19已迁移到nft：
apk add iptables iptables-legacy
rm /sbin/iptables && ln -s /sbin/iptables-legacy /sbin/iptables
rm /sbin/ip6tables && ln -s /sbin/ip6tables-legacy /sbin/ip6tables

4.部分使用nftable的系统上需要设置防火墙模式
在 /etc/default/tailscaled 配置文件中添加变量 TS_DEBUG_FIREWALL_MODE=iptables或nftables或auto
