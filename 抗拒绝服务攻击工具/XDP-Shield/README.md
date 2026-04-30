1. 项目简介（XDP-Shield 是什么）
XDP-Shield 是一个基于 Linux XDP/eBPF 的高性能防护工具：把过滤逻辑挂载到网卡的 XDP 入口，在数据包进入协议栈之前完成匹配与处置，从而在 DDoS/洪泛 场景下以低延迟、高吞吐方式实现丢弃/缓解。

本仓库复现并演示了两类能力：

基于协议/端口的过滤（例如 UDP 5683）
基于阈值的触发式缓解（低速放行、高速超过阈值后开始丢弃）
2. 环境要求
Linux（推荐 Ubuntu/Debian 系；本文示例在 WSL2 Ubuntu 上验证）
clang/llvm、libelf、libconfig、bpftool、libxdp/libbpf、make、tcpreplay、tcpdump
需要 sudo 权限
3. 安装依赖（Ubuntu/Debian）
sudo apt update
sudo apt install -y git libconfig-dev llvm clang libelf-dev build-essential libpcap-dev m4 gcc-multilib
sudo apt install -y tcpreplay tcpdump
WSL2 上常见情况：linux-tools-$(uname -r) 找不到包，可用通用包安装 bpftool：

sudo apt install -y linux-tools-common linux-tools-generic
验证 bpftool（若系统 bpftool 被包装脚本拦截，直接用真实二进制路径）：

/usr/lib/linux-tools/*/bpftool version
4. 构建与安装（示例使用原项目脚本）
chmod +x install.sh
./install.sh --libxdp
安装后一般会得到：

/usr/bin/xdpfw
/usr/bin/xdpfw-add
/usr/bin/xdpfw-del
/etc/xdpfw/xdp_prog.o
/etc/xdpfw/xdpfw.conf
5. 关键编译期开关（阈值触发相关）
为使 ip_pps 等阈值功能按预期工作，需要在编译期开启源 IP 维度限速开关：

文件：src/common/config.h

把下面这一行从注释改为启用：

//#define ENABLE_RL_IP
改为：

#define ENABLE_RL_IP
改完重新安装：

./install.sh
6. 运行前：挂载 bpffs（用于 pinned maps）
如果你看到 “No bpffs found at /sys/fs/bpf” 或 pin map 失败，执行：

sudo mkdir -p /sys/fs/bpf
sudo mount -t bpf bpf /sys/fs/bpf
7. 演示准备：用 veth 构造“攻击端→受害端”（单机可复现）
为了保证 XDP 在 RX 路径上稳定看到回放流量，使用一对 veth 接口：

sudo ip link add vethatk type veth peer name vethvic
sudo ip addr add 10.10.0.1/24 dev vethatk
sudo ip addr add 10.10.0.2/24 dev vethvic
sudo ip link set vethatk up
sudo ip link set vethvic up
8. 演示数据：回放 pcap 并修复二层 MAC（非常关键）
我们用 tcpreplay 回放 pcap 来模拟攻击流量。若 pcap 的以太网 MAC 与 veth 不匹配，包会出现在 vethatk，但不会进入 vethvic RX。

先把 pcap 的目标 IP 重写到 10.10.0.2（示例：原 pcap 目标为 192.168.1.5）：
tcprewrite \
  --infile=/path/to/udp_flood_original.pcap \
  --outfile=/home/user19712/pcap-work/udp_flood_to_veth.pcap \
  --dstipmap=192.168.1.5:10.10.0.2
再把二层源/目的 MAC 重写为 veth 的 MAC（以下 MAC 为示例，按你的实际输出填写）：
ip -br link show vethatk
ip -br link show vethvic
然后：

tcprewrite \
  --infile=/home/user19712/pcap-work/udp_flood_to_veth.pcap \
  --outfile=/home/user19712/pcap-work/udp_flood_to_veth_l2fix.pcap \
  --enet-smac=0a:44:04:01:fe:63 \
  --enet-dmac=62:3c:3a:c0:14:c7 \
  --fixcsum
可选验证（卸载 XDP 后抓包验证链路）：
sudo ip link set vethvic xdp off 2>/dev/null || true
sudo timeout 2 tcpreplay --intf1=vethatk --pps=500 /home/user19712/pcap-work/udp_flood_to_veth_l2fix.pcap
sudo timeout 2 tcpdump -i vethvic -nn "udp and dst port 5683"
9. 核心演示：阈值触发式缓解（XDP-Shield 的“真实”展示）
目标：低速不丢，高速超过阈值才丢。

9.1 写入阈值配置（示例：UDP 5683 + ip_pps=1000）
sudo tee /etc/xdpfw/xdpfw.conf > /dev/null <<'EOF'
verbose = 2;
log_file = "";
interface = "vethvic";
pin_maps = true;
update_time = 0;
no_stats = false;
stats_per_second = true;
stdout_update_time = 1000;
filters = (
  {
    enabled = true,
    action = 0,
    block_time = 30,
    udp_enabled = true,
    udp_dport = 5683,
    ip_pps = 1000
  }
);
EOF
9.2 清状态并启动（避免历史封禁/计数影响）
sudo rm -rf /sys/fs/bpf/xdpfw
sudo ip link set vethvic xdp off 2>/dev/null || true
sudo xdpfw -c /etc/xdpfw/xdpfw.conf
运行后应看到 Attached XDP program to interface 'vethvic'...，并周期性输出 Allowed/Dropped/Passed ... PPS。

9.3 低速基线（应基本不丢）
sudo timeout 5 tcpreplay --intf1=vethatk --pps=500 --stats=1 /home/user19712/pcap-work/udp_flood_to_veth_l2fix.pcap
预期：Passed ≈ 500 PPS，Dropped ≈ 0

9.4 高速攻击（应触发丢弃）
sudo timeout 10 tcpreplay --intf1=vethatk --pps=20000 --stats=1 /home/user19712/pcap-work/udp_flood_to_veth_l2fix.pcap
预期：Dropped 显著上升（接近回放 pps），Passed 显著下降。

10. 定量证据：读取 pinned map 的累计计数（用于报告/录屏）

sudo /usr/lib/linux-tools/*/bpftool map dump pinned /sys/fs/bpf/xdpfw/map_stats \
| python3 -c 'import sys,json; d=json.load(sys.stdin)[0]["values"];
a=sum(v["value"]["allowed"] for v in d);
dr=sum(v["value"]["dropped"] for v in d);
p=sum(v["value"]["passed"] for v in d);
print(f"allowed_total={a} dropped_total={dr} passed_total={p}")'