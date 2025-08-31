tools.md家庭网络一键本地排障脚本 技术方案（MVP）
目标：在本机执行一个脚本，尽可能自动化地定位“家里网络偶尔变慢”的常见根因，输出结构化结果与明确修复建议。脚本优先覆盖“本地后端可做”的检测；路由器/VPS能力作为可选扩展。

适用系统：Windows 10+/macOS 12+/Linux（现代发行版）

注意：本文档包含

检测流程与决策树
每步的目的、命令、判定方法（阈值）
对应修复方案
依赖与权限、超时与安全
输出JSON结构示例
1. 范围与目标
覆盖层面：终端/系统 → LAN/Wi‑Fi → 运营商/WAN → DNS → IPv4/IPv6 → MTU/PMTUD → HTTP各阶段 → 带宽与缓冲膨胀 → 代理/VPN → 复杂网页的资源级别归因（可选）
不覆盖或弱覆盖：光猫PON光功率（通常不可编程）、运营商内部专有指标（需报修配合）
2. 架构与依赖
开发语言建议

Go（跨平台单文件，易并发与超时）或 Python（开发快）
可选组件

Headless Chrome + CDP（Puppeteer/pyppeteer）用于“资源级别”分析
Clash REST（若本机或路由器上运行，并可本地访问）用于代理连接归因
外部命令依赖

通用：curl、iperf3（可选）、mtr 或 tracert/pathping、dig/kdig、ping/fping、tcpdump/libpcap（可选）
Windows：PowerShell（Get‑NetAdapter/Get‑NetRoute）、netsh、pathping
macOS：brew 安装 mtr/iperf3/kdig（airport 命令自带）
Linux：大多已内置；如需 fping/mtr/kdig/iperf3 请安装
3. 总体检测流程（决策树摘要）
环境基线：系统/接口/默认路由/DNS/代理/VPN
LAN基线：ping 网关 → 若异常，优先修 LAN/Wi‑Fi/终端
WAN基线：mtr/ping 运营商常用目标（1.1.1.1/223.5.5.5） → 判定是否早期丢包/抖动（运营商侧）
DNS健康：系统DNS vs 公共DNS 解析时延与落点 → 是否DNS慢/选路不佳
IPv4/IPv6对比：curl -4/-6 → 是否仅v6慢
Path MTU：DF ping 探测最大不分片包 → 判断PMTUD/MSS问题
HTTP阶段：curl -w → 分解为 lookup/connect/tls/ttfb/total → 确认是哪一阶段慢
带宽与缓冲膨胀（可选）：iperf3 + 心跳RTT → 是否SQM问题
代理/VPN与分流：是否走TUN/显式代理；路由表/Clash连接 → 是否误代理/落地差/MTU过小
复杂网页（可选）：CDP抓取每个子资源的远端IP与连接 → 标注是否走VPN/哪条接口 → 找出“拖慢资源”
生成结论与修复建议（verdict）
4. 详细检测步骤
以下每步包含：目的 → 命令 → 判定 → 修复建议。

4.1 环境基线（系统/接口/DNS/路由/代理/VPN）
目的
获取默认网关、出口接口、DNS列表、是否存在代理/VPN（TUN/TAP/Wintun）
命令
默认路由/出口
Linux: ip route get 1.1.1.1
macOS: route -n get 1.1.1.1
Windows: Get-NetRoute -DestinationPrefix 1.1.1.1/32 | sort RouteMetric | select -First 1
接口与链路
Linux: ip -s link
macOS: ifconfig
Windows: Get-NetAdapter | fl, Get-NetAdapterStatistics
Wi‑Fi细节（若为无线）
Linux: iw dev wlan0 link
macOS: /System/Library/PrivateFrameworks/Apple80211.framework/.../airport -I
Windows: netsh wlan show interfaces
DNS服务器
Linux/macOS: scutil --dns（mac）或 resolvectl status（systemd）
Windows: Get-DnsClientServerAddress
代理/VPN
环境变量：env | grep -i proxy
Windows 系统代理：netsh winhttp show proxy；Internet选项（可用PowerShell/注册表）
TUN设备：ip addr | grep -E 'tun|tap'（Win: Get-NetAdapter | where Name -match 'tun|tap|wintun'）
判定
存在TUN/Wintun且默认路由经该接口 → “默认走VPN”
设置显式系统代理（PAC/HTTP/HTTPS/Socket） → “请求可能经本地代理”
修复
后续检测中做A/B：临时关闭代理/VPN，或切直连再重测，观察差异
4.2 LAN基线（到网关）
目的
确认LAN/Wi‑Fi是否稳定（首要排除项）
命令
Linux/macOS: fping -c 50 -q <网关IP> 或 ping -c 50 <网关IP>
Windows: ping -n 50 <网关IP>
判定（有线期望更严）
中位RTT（med）> 10 ms（有线）或 > 30 ms（Wi‑Fi）为异常
抖动（p95-med）> 20 ms（有线）或 > 40 ms（Wi‑Fi）为异常
丢包 > 1% 异常
修复
有线：更换网线/端口，检查协商速率/双工（ethtool/Windows适配器状态）
Wi‑Fi：优先5GHz/6GHz，调整信道（非DFS）、带宽40/80MHz、更新驱动、远离USB3干扰；必要时用网线复测
4.3 WAN基线（到运营商/公共IP）
目的
判断问题是否在运营商早期网络
命令
mtr 或 traceroute + ping
Linux/macOS: mtr -j -c 100 1.1.1.1 与 223.5.5.5
Windows: pathping 1.1.1.1 / pathping 223.5.5.5（或tracert + Ping组合）
判定
早期跳（第2-3跳）开始出现持续丢包/高抖动 → 运营商接入/BRAS问题
仅目标最后一跳显示丢包但中间正常 → 可能为“ICMP限速”非真实丢包
修复
收集mtr/pathping结果报修；同时记录光猫/PPPoE重连日志（若可取）
4.4 DNS健康
目的
排除“ping快但网页慢”的DNS瓶颈与错误选路
命令
系统DNS（逐个）与公共DNS对比（223.5.5.5、1.1.1.1、8.8.8.8）
kdig @ example.com +timeout=1 +tries=1 +stats
判定
lookup时延 > 200 ms（本地/国内站）或 > 400 ms（海外） → DNS慢
返回IP归属ASN/地理明显偏远（用 mmdb/GeoIP 查询） → 解析选路异常
修复
改为就近递归（运营商DNS + 223.5.5.5/1.1.1.1组合）；建议本地缓存（Unbound/SmartDNS）
避免全站走海外DoH导致解析慢；必要时为关键域名设定IP直连/EDNS优化
4.5 IPv4/IPv6对比
目的
判断是否单独的v6劣化或双栈优先级问题
命令
curl -4/-6 -w '{"lookup":%{time_namelookup},"connect":%{time_connect},"tls":%{time_appconnect},"ttfb":%{time_starttransfer},"total":%{time_total},"code":%{http_code}}\n' -o /dev/null -s https://<目标>
判定
仅v6指标显著劣化或失败 → v6路径/前缀/RA问题
修复
临时禁用v6验证；排查路由器v6 PD/RA；或与运营商沟通v6路由质量
4.6 Path MTU/PMTUD
目的
发现“ping快网页慢”的PMTUD黑洞/小MTU问题（PPPoE/VPN常见）
命令
Linux: ping -M do -s 1472 1.1.1.1（失败则逐步减小）
Windows: ping -f -l 1472 1.1.1.1（同上）
macOS：建议安装 tracepath 或使用 hping3；或换目标/方法
判定
最大不分片payload < 1472（以太网典型） → 存在小MTU链路
若使用VPN，阈值可能在 1380~1420
修复
PPPoE：在路由器启用MSS Clamp至 1452 附近，WAN MTU设 1492（设备支持时）
VPN/Clash TUN：将客户端/隧道 MTU 设 1380~1400，并在隧道处做MSS Clamp
4.7 HTTP阶段耗时（分解瓶颈）
目的
明确慢在：DNS、TCP建连、TLS握手、服务器处理/首包、还是全链路
命令
curl -w 同上，对2~3个国内站与1~2个海外站执行（支持URL配置）
判定参考阈值（常见宽带）
lookup：本地/国内 < 80 ms；海外 < 200 ms
connect：本地/国内 < 120 ms；海外 < 350 ms
tls：本地/国内 < 200 ms；海外 < 500 ms
ttfb：本地/国内 < 400 ms；海外 < 800 ms（业务不同差异大）
修复
lookup高 → DNS优化（见4.4）
connect/tls高且MTU异常 → 先修MTU/MSS；或运营商中段拥塞
ttfb高 → 服务器/CDN问题或被错误走VPN；结合4.9判断是否误代理
4.8 带宽与缓冲膨胀（Bufferbloat，可选）
目的
在高上/下载时延骤增，网页体感慢（上传饱和最常见）
命令
iperf3（上/下行各15秒，-P 4 并发）
下行：iperf3 -J -P 4 -t 15 -R
上行：iperf3 -J -P 4 -t 15
同时每100ms ping/fping 网关与1.1.1.1，记录RTT曲线
判定
负载期间 RTT 中位抬升
< 20 ms：A级（优秀）
20~60 ms：B级（可接受）
60 ms：C级（明显缓冲膨胀）

修复
在路由器启用SQM（CAKE/FQ‑CoDel），带宽设为实测的85~90%，启用 dual-srchost/dsthost
控制上传/云备份/监控回看峰值
4.9 代理/VPN与分流判定
目的
识别是否“误代理/落地差/小MTU”导致特定站点慢
命令与方法
路由与出口接口
Linux/macOS: ip route get <目标IP>
Windows: Get-NetRoute 选最小 RouteMetric
检查是否通过 TUN/Wintun 出口（接口名检测或pcap抓SYN包接口）
显式代理模式的盲点：浏览器直连127.0.0.1，需结合代理侧
若本机运行 Clash：查询 http://127.0.0.1:9090/connections（带授权）
判定
目标流量经 TUN/Wintun 或 Clash标记为“被代理”，同时该站点慢 → 判为“代理/分流相关”
修复
切直连重测；按域名生成 DIRECT 规则例外；调整落地节点；降低隧道MTU至 1380~1400 并MSS Clamp
4.10 复杂网页资源级别分析（可选）
目的
发现“主页面直连，某个子资源被代理/走VPN导致拖慢”的场景
方法
用 Headless Chrome + CDP 打开URL，订阅 Network.* 事件，拿到：
每个请求：URL、initiator、connectionId、protocol、remoteIPAddress、timing
对每个远端IP执行“ip route get”判定出口接口；若有Clash，抓取 /connections 匹配域名/规则/节点
判定
在同一页面中，某“连接”被判定为VPN出口，且其资源耗时显著高 → 标记为“拖慢连接”，列出域名/节点
修复
输出建议规则：把相关域名加入直连，或为该域名指定更优节点；若H3在VPN上差，建议强制H2测试
5. 判定与修复建议（规则库摘要）
LAN异常（网关RTT高/抖动/丢包）
Wi‑Fi：切5/6GHz、换信道、减带宽、更新驱动、调整摆放/天线、用网线复测
有线：更换网线/端口、检查WAN/LAN协商速率/错误计数
运营商侧问题（早期跳抖动/丢包）
附mtr/pathping结果报修；保留不同时间段样本
DNS慢/错选路
改上游为就近DNS；本地启用缓存；避免海外DoH；对关键域名配置就近IP
PMTUD/MTU异常
PPPoE：MSS Clamp至1452；WAN MTU 1492
VPN：TUN MTU 1380~1400 + MSS Clamp
IPv6路径问题
临时禁v6验证；修RA/PD；或联系运营商
缓冲膨胀
开SQM（CAKE/FQ‑CoDel），带宽设实测85~90%；优化家庭大上传应用
代理/分流问题
切直连对比；修分流规则（国内域名直连）；更换落地；调低隧道MTU；对问题域名生成DIRECT例外
服务器/CDN慢（仅特定站点慢且直连）
换时间段、换解析（不同CDN边缘）、联系服务商
6. 输出与可视化
脚本输出统一JSON，前端可渲染趋势图与“红黄绿”状态
JSON结构示例（节选）
{
  "system": {
    "os": "Windows 11",
    "defaultGateway": "192.168.1.1",
    "defaultIface": "Ethernet0",
    "dns": ["192.168.1.1","223.5.5.5"],
    "proxy": {"http":"", "https":"", "pac":"off"},
    "vpn": {"tunIfaces":["Wintun"], "defaultViaTun": false}
  },
  "lan": {"rtt_ms":{"med":2.1,"p95":4.8},"loss_pct":0},
  "wan": {
    "mtr": {"target":"1.1.1.1","earlyHopLoss":0,"jitter_ms":8.2}
  },
  "dns": [
    {"server":"192.168.1.1","lookup_ms":25,"answer":["1.2.3.4"]},
    {"server":"223.5.5.5","lookup_ms":18,"answer":["1.2.3.4"]}
  ],
  "mtu": {"max_payload":1472,"suggest_mss":1452},
  "http": [
    {"url":"https://example.com","ipver":"v4","lookup":20,"connect":45,"tls":80,"ttfb":160,"total":350,"code":200}
  ],
  "speedtest": {
    "server":"vps.example.com","down_mbps":920,"up_mbps":45,
    "rtt_bloat_ms":{"down":18,"up":95}
  },
  "proxy": {"routes":[{"dst":"93.184.216.34","iface":"Ethernet0","via":"direct"}]},
  "verdict": [
    {"level":"info","msg":"网络总体正常"},
    {"level":"warn","msg":"上行期间RTT抬升95ms，建议在路由器启用SQM（CAKE）"}
  ]
}
7. 实施细节与工程要点
超时与重试
每个外部命令设定超时（3~10秒级），失败重试不超过1次；全流程期望在60~120秒完成（不含iperf）
并发策略
LAN/WAN/DNS/HTTP可并发采集；iperf3时串行上下行但并发发流；心跳RTT采集独立线程
跨平台适配
Windows优先PowerShell指令并解析JSON（ConvertTo‑Json）；缺少mtr时用pathping替代
macOS通过brew安装缺失组件；airport命令路径固定
安全
严格命令白名单与参数校验；限制并发与调用频次；不执行任意用户命令
后端仅本地监听（127.0.0.1）或启用令牌
日志与“报修包”
保留原始命令输出（裁剪隐私），压缩导出，便于报修或复现
8. 可选扩展（下一步）
路由器采集（SSH或轻探针）：CPU/内存/conntrack、WAN速率/错误、PPPoE/SQM状态、mtr到运营商首跳
VPS外部探针：iperf3、HTTP/3服务、mtr反向观测、STUN/TURN（WebRTC RTT与NAT类型）
复杂网页CDP分析：对指定URL产出“资源-连接-出口-代理”关联报告，并生成Clash DIRECT例外清单
9. 附录：常用命令示例
curl阶段耗时（通用）
curl -4 -w '{"lookup":%{time_namelookup},"connect":%{time_connect},"tls":%{time_appconnect},"ttfb":%{time_starttransfer},"total":%{time_total},"code":%{http_code}}\n' -o /dev/null -s https://example.com
mtr JSON（Linux/macOS）
mtr -j -c 100 1.1.1.1
pathping（Windows）
pathping -n 1.1.1.1
MTU探测
Linux: ping -M do -s 1472 1.1.1.1
Windows: ping -f -l 1472 1.1.1.1
iperf3
iperf3 -J -P 4 -t 15 -R （下行）
iperf3 -J -P 4 -t 15 （上行）
路由查询
Linux/macOS: ip/route -n get 或 route -n get
Windows: Get-NetRoute -DestinationPrefix /32 | sort RouteMetric | select -First 1
Wi‑Fi状态
Windows: netsh wlan show interfaces
macOS: airport -I
Linux: iw dev wlan0 link
DNS
kdig @223.5.5.5 example.com +timeout=1 +tries=1 +stats
10. 示例“修复动作”文案库
开启SQM（CAKE）：在路由器安装 sqm-scripts，下载/上传目标速率设为本次iperf3结果的85~90%，qdisc选cake，nat on，dual-srchost/dsthost on
PPPoE MSS Clamp：在防火墙或WAN接口处开启 TCP MSS clamp to 1452；确认WAN MTU=1492（设备支持）
VPN MTU：将TUN/Clash MTU设为 1380~1400；对TUN的出接口做MSS Clamp；重测
DNS：上游DNS改为“运营商DNS + 223.5.5.5/1.1.1.1”并启本地缓存；避免使用远距DoH；对关键域名生成hosts或SmartDNS策略
Wi‑Fi：切换5/6GHz、固定36/40/44/48信道、带宽40/80MHz、更新网卡驱动、远离USB3干扰；Mesh环境测试仅连主AP
代理分流：国内常用域名走DIRECT；问题域名加入直连例外；更换落地或切换协议；必要时限制大流量应用不走代理
