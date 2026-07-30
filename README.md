#基于 eBPF 的AI智能网络监控系统

基于 eBPF 的AI智能网络监控系统是一个从零构建的 Linux 实时网络监控系统，利用 **eBPF 内核探针** 实现零开销的流量采集，通过 **D-Bus** 完成跨进程事件通知，提供网卡状态、延迟、丢包、信号强度、流量等多维度网络指标的实时监控与质量评估。

---

## 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                        NetPulse Server                       │
│                                                             │
│  ┌──────────┐  ┌───────────┐  ┌───────┐  ┌──────────────┐  │
│  │ iface    │  │ RTT       │  │ TCP   │  │ Traffic      │  │
│  │ monitor  │  │ monitor   │  │ Loss  │  │ Analyzer     │  │
│  │ (10s)    │  │ (10s)     │  │ (10s) │  │ (10s)        │  │
│  └────┬─────┘  └─────┬─────┘  └───┬───┘  └──────┬───────┘  │
│       │              │           │              │          │
│  ┌────▼──────────────▼───────────▼──────────────▼───────┐  │
│  │              ServerContext (共享状态)                  │  │
│  │         NetInfo 列表 + mutex 线程安全保护              │  │
│  └────────────────────────┬────────────────────────────┘  │
│                           │                                │
│  ┌────────────────────────▼────────────────────────────┐  │
│  │  NetworkEventManager + DbusService                    │  │
│  │  事件回调 → D-Bus Signal 发送                          │  │
│  └────────────────────────┬────────────────────────────┘  │
└───────────────────────────│────────────────────────────────┘
                            │ D-Bus
  ┌─────────────────────────▼────────────────────────────────┐
  │           NetPulse Client (libnetpulse.so)                │
  │                                                          │
  │  C API: get_interfaces / health_check / ping_host        │
  │  Event: subscribe_event / check_events / unsubscribe     │
  │  Quality: subscribe_network_quality / check              │
  └──────────────────────────────────────────────────────────┘
```

---

## 核心技术

| 技术 | 用途 | 文件 |
|------|------|------|
| **C++17** | 核心开发语言 | `*.cpp` / `*.hpp` |
| **eBPF kprobe** | 内核态 TCP/UDP 流量采集 | `server/src/flow_rate.bpf.c` |
| **libdbus-1** | 服务端 ↔ 客户端进程间通信 | `dbus_service.cpp` / `client.cpp` |
| **多线程** | 7 个并行监控线程 | `server.cpp` 线程启动 |
| **ICMP 原始套接字** | RTT 延迟探测 | `net_ping.h` / `net_ping.cpp` |
| **/proc/net/snmp** | TCP 丢包率统计 | `net_tcp.h` / `net_tcp.cpp` |
| **wpa_supplicant** | Wi-Fi RSSI 信号监控 | `net_wifiriss.cpp` |
| **Google glog** | 模块化日志系统 | `logger.hpp` |
| **FAISS + 阿里百炼** | AI 辅助网络分析（可选扩展） | `AI-assisted analysis/` |

---

## 快速开始

### 依赖安装

```bash
# Ubuntu / Debian
sudo apt-get update
sudo apt-get install -y build-essential clang llvm pkg-config \
  libdbus-1-dev libglog-dev libelf-dev zlib1g-dev libcap-dev \
  linux-headers-$(uname -r) libbpf-dev
```

### 编译

```bash
make all
```

编译产物：
- `server/bin/netpulse-server` — 服务端
- `client/bin/test-client` — 命令行测试工具
- `client/lib/libnetpulse.so` — 客户端动态库

### 运行

**终端 1 — 启动服务端：**
```bash
./server/bin/netpulse-server
```

**终端 2 — 客户端测试：**
```bash
# 查看网卡信息
make test-client COMMAND=get

# 健康检查
make test-client COMMAND=health

# Ping 测试
make test-client COMMAND="ping google.com"

# 监听事件
make test-client COMMAND=subscribe

# 运行全部测试
make test-client COMMAND=all
```

---

## 监控指标

| 指标 | 采集方式 | 精度 |
|------|----------|------|
| 网卡状态 | getifaddrs + 路由表 | 实时 |
| RTT 延迟 | ICMP Echo (原始套接字) | 毫秒级 |
| TCP 丢包率 | /proc/net/snmp 统计 | 百分比 |
| Wi-Fi RSSI | wpa_supplicant ctrl 接口 | dBm |
| 流量速率 | eBPF kprobe → LRU_HASH MAP | Bps / pps |
| 活跃连接数 | eBPF 五元组追踪 | 65536 条上限 |
| 网络质量评分 | RTT/丢包/RSSI/流量加权 | 0-100 分 |

---

## 事件系统

NetPulse 支持 6 种网络事件，通过 D-Bus Signal 主动推送给客户端：

| 事件类型 | 说明 | 优先级 |
|----------|------|--------|
| `InterfaceChanged` | 网卡添加/移除 | 8 |
| `ConnectionModeChanged` | 上网网卡切换 | 9（最高） |
| `NetworkQualityChanged` | 网络质量等级变化 | 7 |
| `TcpLossRateChanged` | TCP 丢包率变化 | 6 |
| `RttChanged` | RTT 延迟变化 | 5 |
| `RssiChanged` | Wi-Fi 信号变化 | 4 |

---

## 客户端集成

```c
#include "client/weaknet_client.h"

int main() {
    weaknet_init();

    char buf[4096], err[256];
    weaknet_get_interfaces(buf, sizeof(buf), err, sizeof(err));
    printf("Interfaces: %s\n", buf);

    char result[512];
    weaknet_ping_host("google.com", result, sizeof(result), err, sizeof(err));
    printf("Ping: %s\n", result);

    weaknet_cleanup();
    return 0;
}
```

编译：`g++ -o app main.cpp -Lclient/lib -lnetpulse`

---

## 目录结构
```
├── AI-assisted analysis
│   ├── __pycache__
│   │   ├── interactive_rag.cpython-310.pyc
│   │   ├── local_vector_rag_analyzer.cpython-310.pyc
│   │   ├── network_diagnosis_demo.cpython-310.pyc
│   │   ├── network_diagnosis_tool.cpython-310.pyc
│   │   ├── network_knowledge_base.cpython-310.pyc
│   │   ├── network_rag_system.cpython-310.pyc
│   │   ├── network_rag_time_analysis.cpython-310.pyc
│   │   ├── optimized_network_rag.cpython-310.pyc
│   │   └── simple_rag_analyzer.cpython-310.pyc
│   ├── install_rag.sh
│   ├── interactive_rag.py
│   ├── local_vector_rag_analyzer.py
│   ├── log_capture.py
│   ├── network_knowledge_base.py
│   ├── network_knowledge_vectorstore_local
│   │   ├── index.faiss
│   │   └── index.pkl
│   ├── optimized_network_rag.py
│   ├── requirements.txt
│   ├── simple_rag_analyzer.py
│   ├── test_dashscope_api.py
│   ├── true_rag_analyzer.py
│   └── vector_rag_analyzer.py
├── Makefile
├── README.md
├── client
│   ├── client.cpp
│   ├── example_usage.cpp
│   ├── ping_example.cpp
│   ├── test_client
│   ├── test_client.cpp
│   ├── test_network_quality.cpp
│   └── weaknet_client.h
├── config.mk
├── install.sh
├── logs
│   ├── config
│   │   └── glog.conf
│   └── server
├── server
│   ├── Makefile
│   ├── include
│   │   ├── common.hpp
│   │   ├── dbus_service.hpp
│   │   ├── event_manager.hpp
│   │   ├── logger.hpp
│   │   ├── looper.hpp
│   │   ├── net_iface.h
│   │   ├── net_info.hpp
│   │   ├── net_ping.h
│   │   ├── net_tcp.h
│   │   ├── net_traffic.h
│   │   ├── net_wifiriss.h
│   │   ├── network_quality_assessor.hpp
│   │   ├── rssi_monitor.hpp
│   │   ├── rtt_monitor.hpp
│   │   ├── serializer.hpp
│   │   ├── server.hpp
│   │   ├── tcp_loss_monitor.hpp
│   │   ├── traffic_analyzer.hpp
│   │   ├── traffic_anomaly_detector.h
│   │   ├── using_iface.h
│   │   └── weak_netmgr.hpp
│   └── src
│       ├── dbus_service.cpp
│       ├── event_manager.cpp
│       ├── flow_rate.bpf.c
│       ├── logger.cpp
│       ├── looper.cpp
│       ├── main.cpp
│       ├── net_iface.cpp
│       ├── net_info.cpp
│       ├── net_ping.cpp
│       ├── net_tcp.cpp
│       ├── net_traffic.cpp
│       ├── net_wifiriss.cpp
│       ├── network_quality_assessor.cpp
│       ├── rssi_monitor.cpp
│       ├── rtt_monitor.cpp
│       ├── serializer.cpp
│       ├── server.cpp
│       ├── tcp_loss_monitor.cpp
│       ├── traffic_analyzer.cpp
│       ├── traffic_anomaly_detector.cpp
│       ├── using_iface.cpp
│       ├── vmlinux.h
│       └── weak_netmgr.cpp
```
---

## AI 辅助分析

`AI-assisted analysis/` 目录包含可选的 Python 工具集，使用 **阿里云百炼（DashScope）** 平台的 `qwen-plus` 模型进行网络日志的智能分析，并基于 FAISS 向量库实现 RAG（检索增强生成）。

**提示：** 调用阿里百炼 API 是 **付费服务**。`qwen-plus` 模型按 token 计费（输入约 ¥2/百万 tokens，输出约 ¥6/百万 tokens，价格可能浮动）。需要自行配置 API 密钥才能使用。如果不配置 API 密钥，该部分功能不会产生任何费用。

---

## License

MIT
