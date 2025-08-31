# 家庭网络排障脚本详细代码方案

## 1. 检测流程和实现方案

### 1.1 环境基线检测
**检测内容：**
- 系统信息、默认网关、出口接口
- DNS服务器列表
- 代理/VPN状态

**使用工具：**
- `subprocess`执行系统命令
- Windows: PowerShell `Get-NetRoute`, `Get-DnsClientServerAddress`
- macOS/Linux: `ip route`, `scutil`, `resolvectl`

**结果分析：**
- 识别默认网络接口
- 获取DNS服务器地址
- 检测是否存在代理或VPN连接

**解决方案：**
- 记录环境信息供后续分析
- 标记代理/VPN状态用于后续对比测试

### 1.2 LAN基线检测（到网关）
**检测内容：**
- 到网关的ping测试（50次）
- 分析RTT中位数、抖动、丢包率

**使用工具：**
- `subprocess`执行ping命令
- `statistics`模块进行数据分析

**结果分析：**
- 有线网络：RTT > 10ms、抖动 > 20ms、丢包 > 1%为异常
- 无线网络：RTT > 30ms、抖动 > 40ms、丢包 > 1%为异常

**解决方案：**
- 有线：建议更换网线/端口，检查协商速率
- 无线：建议切换5GHz/6GHz，调整信道，更新驱动

### 1.3 WAN基线检测（到运营商/公共IP）
**检测内容：**
- 到1.1.1.1和223.5.5.5的mtr/pathping测试
- 分析早期跳的丢包和抖动

**使用工具：**
- `subprocess`执行mtr或pathping命令
- JSON解析mtr输出

**结果分析：**
- 早期跳（第2-3跳）持续丢包/高抖动表示运营商问题
- 仅最后一跳丢包可能是ICMP限速

**解决方案：**
- 收集mtr结果报修运营商
- 记录光猫/PPPoE重连日志

### 1.4 DNS健康检测
**检测内容：**
- 系统DNS与公共DNS的解析时延对比
- 解析结果的ASN/地理位置

**使用工具：**
- `subprocess`执行dig/kdig命令
- `requests`查询IP地理信息API

**结果分析：**
- 解析时延 > 200ms（国内）或 > 400ms（海外）为慢
- 解析IP地理位置偏远表示选路异常

**解决方案：**
- 建议使用就近递归DNS（运营商DNS + 223.5.5.5/1.1.1.1）
- 建议本地缓存DNS（Unbound/SmartDNS）

### 1.5 IPv4/IPv6对比检测
**检测内容：**
- IPv4和IPv6的HTTP阶段耗时对比

**使用工具：**
- `subprocess`执行curl命令
- JSON解析curl输出

**结果分析：**
- 仅IPv6指标显著劣化表示IPv6路径问题

**解决方案：**
- 临时禁用IPv6验证
- 排查路由器IPv6配置

### 1.6 Path MTU/PMTUD检测
**检测内容：**
- 不同大小数据包的ping测试

**使用工具：**
- `subprocess`执行ping命令（带DF标志）

**结果分析：**
- 最大不分片payload < 1472表示存在小MTU链路

**解决方案：**
- PPPoE：路由器启用MSS Clamp至1452，WAN MTU设1492
- VPN：TUN MTU设1380~1400，并做MSS Clamp

### 1.7 HTTP阶段耗时检测
**检测内容：**
- 对2-3个国内站和1-2个海外站的HTTP阶段耗时分析

**使用工具：**
- `subprocess`执行curl命令
- JSON解析curl输出

**结果分析：**
- 各阶段耗时阈值判断瓶颈位置

**解决方案：**
- 根据瓶颈位置采取相应修复措施

### 1.8 带宽与缓冲膨胀检测（可选）
**检测内容：**
- iperf3上下行测试
- 同时监控ping RTT变化

**使用工具：**
- `subprocess`执行iperf3命令
- 并发执行ping监控

**结果分析：**
- 负载期间RTT中位抬升判断缓冲膨胀程度

**解决方案：**
- 路由器启用SQM（CAKE/FQ-CoDel）
- 控制上传应用峰值

### 1.9 代理/VPN与分流判定
**检测内容：**
- 目标流量出口接口检测
- Clash连接分析（如果可用）

**使用工具：**
- `subprocess`执行路由查询命令
- HTTP请求Clash API（如果可用）

**结果分析：**
- 流量经TUN接口且站点慢表示代理相关问题

**解决方案：**
- 切直连重测
- 生成DIRECT规则例外

## 2. 模块划分

### 2.1 主模块 (main.py)
- 协调各检测模块执行
- 处理命令行参数
- 输出最终结果

### 2.2 系统信息模块 (system_info.py)
- 获取系统信息
- 检测网络接口和路由
- 检测DNS和代理设置

### 2.3 LAN检测模块 (lan_test.py)
- 执行到网关的ping测试
- 分析RTT、抖动、丢包

### 2.4 WAN检测模块 (wan_test.py)
- 执行mtr/pathping测试
- 分析运营商网络质量

### 2.5 DNS检测模块 (dns_test.py)
- 执行DNS解析测试
- 分析解析时延和选路

### 2.6 IPv6检测模块 (ipv6_test.py)
- 执行IPv4/IPv6对比测试
- 分析双栈性能差异

### 2.7 MTU检测模块 (mtu_test.py)
- 执行Path MTU探测
- 分析MTU问题

### 2.8 HTTP检测模块 (http_test.py)
- 执行HTTP阶段耗时测试
- 分析各阶段性能

### 2.9 带宽检测模块 (bandwidth_test.py)
- 执行iperf3测试
- 分析缓冲膨胀

### 2.10 代理检测模块 (proxy_test.py)
- 检测代理/VPN状态
- 分析流量出口

### 2.11 结果分析模块 (analyzer.py)
- 汇总各模块检测结果
- 生成诊断结论和修复建议

### 2.12 工具模块 (utils.py)
- 提供通用工具函数
- 跨平台命令执行
- 数据处理和分析