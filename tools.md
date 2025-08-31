 # 系统工具依赖列表

## 通用工具
- curl: HTTP请求和测试
- ping: 网络连通性测试
- dig/kdig: DNS查询工具

## 平台特定工具

### Linux
- iproute2: 网络接口和路由管理
- mtr: 网络路径和质量测试
- iperf3: 网络带宽测试
- fping: 批量ping测试

### macOS
- mtr: 通过brew安装
- iperf3: 通过brew安装
- fping: 通过brew安装
- kdig: 通过brew安装
- airport: Wi-Fi状态查询（系统自带）

### Windows
- PowerShell: 系统信息查询
- pathping: 网络路径和质量测试
- iperf3: 网络带宽测试
- mtr: 可选安装

## 可选工具
- tcpdump/libpcap: 网络抓包分析
- Headless Chrome + CDP: 复杂网页资源分析
- Clash: 代理连接归因分析