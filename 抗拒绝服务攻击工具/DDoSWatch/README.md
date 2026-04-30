# DDoSWatch 回放流量触发检测与封禁（实现流程）

目标：用本机 veth 闭环 + `tcpreplay` 回放 PCAP，让 DDoSWatch 完成 **检测 → Ban list → 调用处置脚本 → 下发 nft 规则** 的完整链路验证。

> 约束：本文**说明性文字**使用 DDoSWatch 命名；但为保持你的环境一致性，**所有命令/代码块仍使用 `fastnetmon`**（含服务名、配置路径、日志路径等）。

## 1. 启动服务并确认在运行

```bash
sudo systemctl enable --now fastnetmon
systemctl status fastnetmon
journalctl -u fastnetmon -n 200 --no-pager
```

## 2. 确认采集方式与监听接口（只读检查）

```bash
ip link
sudo grep -nE "interface|interfaces|netflow|sflow|mirror|pfring|afpacket|dpdk|ban|iptables|bgp" /etc/fastnetmon*.conf 2>/dev/null
sudo tail -n 200 /var/log/fastnetmon.log | grep -aEi 'AF_PACKET will listen on|mirror|sflow|netflow'
```

## 3. 准备本机回放闭环（veth）

```bash
sudo ip link add veth-replay type veth peer name veth-suricata
sudo ip link set veth-replay up
sudo ip link set veth-suricata up
ip link | grep -E "veth-replay|veth-suricata"
```

用 `tcpdump` 先验证链路通不通（先不管服务）：

```bash
sudo tcpdump -ni veth-suricata -c 20
```

另开终端回放一次（任意小 PCAP 都行）：

```bash
sudo tcpreplay -i veth-replay --pps=50000 "$PCAP"
```

## 4. 让 DDoSWatch 监听回放对端接口

把监听接口指向 `veth-suricata`（注意：如果配置里出现多次，确保**最后生效**的那处也改了），改完重启并确认日志显示监听该接口：

```bash
sudo grep -n "interfaces" /etc/fastnetmon.conf
sudo systemctl restart fastnetmon
sudo tail -n 200 /var/log/fastnetmon.log | grep -aEi 'AF_PACKET will listen on'
```

## 5. 让回放流量进入“受保护网段”

检查当前受保护网段（只读）：

```bash
echo "=== /etc/networks_list ==="
sudo cat /etc/networks_list
echo "=== /etc/networks_whitelist ==="
sudo cat /etc/networks_whitelist
```

如果 PCAP 的 IP 不在受保护网段里，需要先用 `tcprewrite` 生成“对齐后的 PCAP”（原始文件不变）。

示例：把源映射到“外部网段”，把目的固定到受保护网段内的单一受害者 `10.10.10.250`：

```bash
REWRITE_PCAP="/tmp/botnet_external_to_victim250.pcap"
sudo tcprewrite \
  --infile="$PCAP" \
  --outfile="$REWRITE_PCAP" \
  --srcipmap=192.168.1.0/24:203.0.113.0/24 \
  --dstipmap=192.168.1.0/24:10.10.10.250/32 \
  --fixcsum
```

（可选）快速验证重写结果（确保 dst 变成 `10.10.10.250`）：

```bash
sudo tshark -r "$REWRITE_PCAP" -n -T fields -e ip.src -e ip.dst | head -n 20
```

## 6. 回放并触发 Ban list

持续回放更容易满足检测窗口（阈值不改的前提下）：

```bash
sudo tcpreplay -i veth-replay --pps=200000 --loop=0 "$REWRITE_PCAP"
```

另开终端观察：

```bash
sudo fastnetmon_client
```

期望现象：`10.10.10.250` 在界面里出现 `banned`，并且 Ban list 有时间戳记录。

## 7. 验证处置脚本与 nft 规则是否下发

先看处置日志（脚本是否被调用）：

```bash
sudo tail -n 50 /var/log/fastnetmon_actions/disposition.log
sudo ls -la /var/log/fastnetmon_actions | tail -n 50
```

再看 nft 规则是否已生成：

```bash
command -v nft
sudo nft list ruleset | sed -n '1,220p'
```

如果系统没有 `nft`，先装工具：

```bash
sudo apt update
sudo apt install -y nftables
```

## 8. 常用排查清单（只保留命令）

```bash
systemctl status fastnetmon --no-pager
journalctl -u fastnetmon -n 200 --no-pager
sudo tail -n 200 /var/log/fastnetmon.log | grep -aEi 'attack|ban|threshold|AF_PACKET|listen'
sudo grep -nE "mirror_afpacket|interfaces\s*=|netflow|sflow" /etc/fastnetmon*.conf 2>/dev/null
ip link
sudo tcpdump -ni veth-suricata -c 20
```
