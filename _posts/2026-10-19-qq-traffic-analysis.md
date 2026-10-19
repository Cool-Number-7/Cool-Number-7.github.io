---
layout: post
title: "QQ 流量分析实验：PC—PC、PC—安卓手机的协议识别与文件恢复"
date: 2026-10-19 00:00:00 +0800
categories: [网络安全, 实验记录]
permalink: /qq-traffic-analysis/
description: "使用 Wireshark 分析两种 QQ 传输场景，依据 TCP 会话和 HTTP 载荷恢复测试文件，并通过 SHA-256 校验。"
---

本次作业要求分析 QQ 在 **PC—PC、PC—手机** 两种场景下的流量，回答“UDP 还是 TCP”“加密还是明文”，并尝试恢复可能以明文传输的文件。本文记录实际抓包结果、恢复过程及证据边界。

**实验结论：两组已定位的测试文件上传会话均使用 TCP，文件正文能从 HTTP 载荷直接提取，恢复结果与原文件逐字节一致。** 其中 PC—PC 会话使用 443 端口，但实际承载明文 HTTP，不能仅根据端口号判断加密。文字消息未获得足够证据，本文不将“搜索不到明文”等同于“已证明加密”。

## 一、实验环境与设计

| 项目 | 实际情况 |
|---|---|
| 抓包端 | Windows 电脑，WLAN 网卡 |
| 分析工具 | Wireshark 4.6.9、配套 TShark / capinfos，Npcap 驱动 |
| 本机 QQ | 文件版本 9.9.21.39038；产品版本 9.9.21.39038-6a73892f |
| PC—手机 | 同一个 QQ 账号在电脑、安卓手机登录，通过设备间传输入口测试 |
| PC—PC | 本机与同学电脑通过 QQ 会话测试；两台电脑连接同一 Wi-Fi |
| 网络条件 | 本机使用过 FlClash；最终 PC—PC 文件补抓在关闭 FlClash 后进行 |
| 未记录项目 | 手机 QQ、安卓系统、同学电脑 QQ 的具体版本，以及手机网络连接方式 |

本机 QQ 版本通过已安装程序的文件元数据读取。未记录的版本不作推测。实验只分析自有设备及同学配合完成的测试通信，不对其他人的会话作分析。

两组场景的账号关系、文件内容、代理状态不同，因此本实验属于行为观察，**不能把所有差异都归因于 PC 与手机的区别**。

### 测试内容

- 文字测试使用专用标识，例如 `QQ_TEXT_TEST_001`、`QQ_PCPC_TEXT_001`。
- PC—手机传输 `qq-test.txt`，57 字节。
- PC—PC 最终实际传输 `QQ_PCPC_TEXT_003.txt`，108 字节。尽管名称含 TEXT，它是一个文件，正文以 `QQ_PCPC_FILE_TEST_001` 开始。
- 接收方在线接收并下载完成后停止抓包，文件保持在本地供校验使用。

## 二、抓包方法与异常处理

1. 安装 Npcap，使 Wireshark 能识别本地网卡。
2. 选择实际联网的 WLAN。初始捕获不设置仅限 TCP、UDP 或对端 IP 的过滤条件，避免漏掉服务器中转流量。
3. 先记录 QQ 空闲流量，再分别进行文字和在线文件传输。
4. 以操作场景、时间、TCP 会话和已知测试内容定位相关数据。单凭“此时 QQ 正在运行”不能认定所有包都属于 QQ。
5. 在 Wireshark 中用显示过滤器筛选、搜索字符串和十六进制内容，并通过“追踪 TCP 流”核对应用层数据。

### FlClash 导致的一次无效捕获

最初 PC—PC 测试同时选择 WLAN、FlClash 及另外两个 WLAN 接口。由于开启 FlClash 时文件无法发送，在发文件前关闭了 FlClash。Wireshark 随后提示虚拟网卡已断开，**整轮捕获自动停止**，文件却是在之后发送的。这轮不能作为成功文件传输的证据。

补抓时只选 WLAN，设置直接写入本地文件和 300 秒自动停止上限。关闭 FlClash 后完成 QQ 操作，再停止捕获并恢复网络连接。实际有效捕获持续约 154 秒。文件名中的 `direct` 只是实验命名，**不表示已证明两台电脑点对点直连**。

### 本地证据清单

| 文件 | 场景 | 数据包数 | 持续时间 |
|---|---|---:|---:|
| 01-pc-mobile-idle.pcapng | PC—手机空闲基线 | 775 | 39.543 秒 |
| 02-pc-mobile-text.pcapng | PC—手机文字 | 7049 | 162.516 秒 |
| 03-pc-mobile-file.pcapng | PC—手机文件 | 14894 | 198.612 秒 |
| 04-pc-pc-text.pcapng | PC—PC 文字，含虚拟网卡 | 13332 | 211.788 秒 |
| 05-pc-pc-file.pcapng | 虚拟网卡断开，中断样本 | 13822 | 95.982 秒 |
| 06-pc-pc-direct.pcapng | 仅 WLAN 的 PC—PC 补抓 | 9471 | 153.750 秒 |

持续时间包括操作准备与等待确认。多网卡捕获可能记录同一通信经过不同接口时的多个副本，不能简单相加解释为业务数据量。**原始 PCAPNG 不公开上传**，以下用脱敏的证据表、正文和校验值展示结果。

## 三、PC—手机：TCP 80 端口上的明文文件上传

在文件实验中搜索 `QQ_FILE_TEST_001`，命中 **第 7987 帧**，所属 **TCP stream 39**，相对时间为 **156.899719 秒**。

| 原始帧号 | 观察结果 | 作用 |
|---|---|---|
| 7981 | SYN | 发起 TCP 连接 |
| 7982 | SYN、ACK | 服务端响应 |
| 7983 | ACK | 完成三次握手 |
| 7984 | TCP 载荷 276 字节 | HTTP 请求头 |
| 7987 | HTTP POST，载荷 218 字节 | 包含测试文件正文 |
| 7994 | HTTP 200 | 服务端响应 |
| 13496、13498、13499 | FIN/ACK 与确认 | 连接结束 |

这条会话连接电脑与公网服务端的 **TCP 80 端口**。HTTP 请求路径为 `/ftn_handler`，查询参数已省略。文件正文直接出现在 HTTP 请求体中，没有对这部分正文提供传输内容保密。

复查过滤器：

```text
tcp.stream == 39
frame.number == 7987
frame contains "QQ_FILE_TEST_001"
```

这些证据确认的是**电脑上传链路**。电脑侧抓包并未完整观察手机向服务器下载的链路，不能断言手机侧采用相同的加密方式。

### 恢复 57 字节文件

HTTP 请求体共 218 字节，已知标识从零基偏移 **161** 开始，到请求体末尾正好 57 字节。标识对应的十六进制为：

```text
51 51 5f 46 49 4c 45 5f 54 45 53 54 5f 30 30 31
 Q  Q  _  F  I  L  E  _  T  E  S  T  _  0  0  1
```

提取到的文件内容：

```text
QQ_FILE_TEST_001
This file is for my Wireshark homework.
```

将 HTTP 请求体相应字节直接写入文件，而不是将屏幕显示的文本重新复制保存，这样可以保留 CRLF 等原始字节。恢复文件与原文件均为 57 字节，逐字节比较一致，SHA-256 均为：

```text
2AD46506A52099DF95C10F833FDA4675415BDCD69339BC4CDB68584E7B6D5275
```

[查看恢复出的测试文件]({{ '/assets/qq-lab/mobile-recovered.txt' | relative_url }})

## 四、PC—PC：443 端口不等于 HTTPS

有效补抓中，实际文件定位于 **第 9207 帧、TCP stream 54**，相对时间为 **136.980259 秒**，目的端口为 **443**。

| 原始帧号 | 观察结果 | 作用 |
|---|---|---|
| 9202、9203、9204 | SYN、SYN/ACK、ACK | TCP 三次握手 |
| 9205 | TCP 载荷 259 字节 | HTTP 请求头 |
| 9207 | HTTP POST，请求体 1269 字节 | 包含文件名和正文 |
| 9209 | HTTP 200 | 服务端响应 |
| 9461、9462、9463 | FIN/ACK 与确认 | 连接结束 |

请求路径为 `/cgi-bin/httpconn`，账号相关查询参数已省略。Wireshark 可以直接解析 HTTP，请求体中出现可读文件内容，并可恢复出与原文件完全一致的字节。该流没有解析出 TLS。

因此，**这次上传实际使用 TCP 上的明文 HTTP，端口号是 443**。判断协议时需要看真实报文，不能将“443”直接替换成“HTTPS”。

复查过滤器：

```text
tcp.stream == 54
frame.number == 9207
frame contains "QQ_PCPC_FILE_TEST_001"
```

虽然双方连接同一 Wi-Fi，这条已定位的上传连接仍面向公网服务端。本次不能据此证明双方直接通信。

### 按应用层长度字段恢复文件

第 9207 帧的 1269 字节 HTTP 请求体具有下列可核对的结构：

| 部分 | 长度 | 本帧内容或含义 |
|---|---:|---|
| 起始字节 | 1 字节 | `0x28` |
| 头部长度字段 | 4 字节 | 大端整数 1151，即 `00 00 04 7f` |
| 数据长度字段 | 4 字节 | 大端整数 108，即 `00 00 00 6c` |
| 应用头部 | 1151 字节 | 含传输元数据，不公开 |
| 文件正文 | 108 字节 | 从零基偏移 1160 开始 |
| 结束字节 | 1 字节 | `0x29` |

长度关系可以相互校验：

```text
1 + 4 + 4 + 1151 + 108 + 1 = 1269
正文起点 = 1 + 4 + 4 + 1151 = 1160
```

恢复时必须排除末尾的 `0x29`。如果把它当作正文保留，会多出一个右括号，文件校验失败。上述解释适用于本帧，不据此推广为所有 QQ 传输的固定格式。

恢复出的实际内容：

```text
QQ_PCPC_FILE_TEST_001
This file is for the PC-to-PC QQ transfer experiment.
Experiment-ID: PCPC-20261019-A
```

原文件为 CRLF 换行、末尾没有换行。恢复文件与桌面原文件均为 **108 字节**，逐字节一致，SHA-256 均为：

```text
8B7ABD1B35189697DDECB0860E2E77FC4449D02C73E57FC95704BE043CEB66EE
```

[查看恢复出的测试文件]({{ '/assets/qq-lab/pcpc-recovered.txt' | relative_url }})

### 可复查的提取方法

安装 Wireshark 后，可用配套 TShark 提取所需 HTTP 请求体。下面的 Python 示例只读本地抓包，输出从载荷恢复的文件：

```python
from pathlib import Path
import hashlib
import subprocess

TSHARK = "tshark"  # 或改为本机 tshark.exe 的完整路径

def http_body(capture, frame):
    result = subprocess.run(
        [TSHARK, "-r", capture, "-Y", f"frame.number == {frame}",
         "-T", "fields", "-e", "http.file_data"],
        check=True, capture_output=True, text=True
    )
    return bytes.fromhex(result.stdout.strip())

mobile = http_body("03-pc-mobile-file.pcapng", 7987)
start = mobile.index(b"QQ_FILE_TEST_001")
assert start == 161 and len(mobile) == 218
mobile_file = mobile[start:]
assert len(mobile_file) == 57

pcpc = http_body("06-pc-pc-direct.pcapng", 9207)
header_len = int.from_bytes(pcpc[1:5], "big")
file_len = int.from_bytes(pcpc[5:9], "big")
assert pcpc[0] == 0x28 and pcpc[-1] == 0x29
assert 9 + header_len + file_len + 1 == len(pcpc)
pcpc_file = pcpc[9 + header_len:9 + header_len + file_len]
assert header_len == 1151 and file_len == 108

for name, data in [
    ("mobile-recovered.txt", mobile_file),
    ("pcpc-recovered.txt", pcpc_file),
]:
    Path(name).write_bytes(data)
    print(name, len(data), hashlib.sha256(data).hexdigest())
```

本例文件正文恰好完整包含在相应帧中。更大的文件可能跨多个 TCP 段，需要先重组、处理重传与分块，不能简单按抓包顺序拼接所有载荷。

## 五、UDP、文字消息与加密判断

本机捕获中同时存在 UDP 与 TCP。以 PC—手机两份抓包的 IPv4 数据为例：

| 抓包 | UDP 包数 | TCP 包数 |
|---|---:|---:|
| 文字场景 | 6565 | 463 |
| 文件场景 | 12808 | 2030 |

**上表统计的是整个网卡上的流量，不是 QQ 专属统计。** 其中可能包含代理、DNS 和其他程序的活动。已通过文件正文和会话定位的两次上传均为 TCP，其他 UDP 流量没有足够证据归属到某项 QQ 操作。

对文字场景逐帧搜索已知标识没有获得独立聊天消息明文的可靠命中。补抓中 `QQ_PCPC_TEXT_003` 出现在文件名里，不能当作文字聊天内容的证据。也不能由“搜索不到”直接推出加密，因为跨段、编码、压缩或定位不充分都会影响结果。

因此对“加密还是明文”的回答要分功能：

- **本次两组测试文件的上传正文：已证实为明文，可完整恢复。**
- **独立文字聊天：本次证据不足，未确定加密方式。**
- **接收端下载链路、其他 QQ 功能：没有完整观察，不作推广。**

## 六、文件头识别与本实验的区别

题目提示可根据不同文件的头部特征识别文件起点。本次实际测试的是 TXT，**TXT 没有统一的固定文件头**，所以使用已知测试标识定位，并结合应用层长度字段确定边界。

本实验没有进行 PNG、PDF 等格式的恢复验证。后续若测试其他格式，应结合格式特征、协议重组、文件结构和哈希校验，而不能只找到一个文件头就宣称恢复成功。本次使用“长度一致 + 逐字节一致 + SHA-256 一致”验证结果。

## 七、作业问题汇总

| 作业问题 | PC—手机 | PC—PC |
|---|---|---|
| UDP or TCP？ | 已定位文件上传使用 TCP | 已定位文件上传使用 TCP |
| 加密 or 明文？ | 上传正文为明文 HTTP | 上传正文为明文 HTTP，目的端口为 443 |
| 能否恢复文件？ | 成功，57 字节 | 成功，108 字节 |
| 关键证据 | 帧 7987，流 39 | 帧 9207，流 54 |
| 是否与原文件一致？ | 逐字节、SHA-256 均一致 | 逐字节、SHA-256 均一致 |
| 文字消息结论 | 证据不足 | 证据不足 |

这次实验说明，协议判断应依据报文结构和具体会话；文件恢复应依据真实字节与校验结果。不能将一个上传会话的结论扩展为整个 QQ 的安全结论，也不能仅凭端口或不可读载荷判断加密。

## 八、隐私处理与资料说明

本文省略真实账号、昵称、私人聊天内容、设备标识、本地用户名路径、真实端点地址和完整请求参数。公开附件仅包含专门构造的无隐私测试文本；原始抓包和含身份信息的应用头部保留在本地。

证据表由本次 Wireshark / TShark 结果整理，保留原始帧号、会话号、长度及校验值，便于在本地复查。正文中的示例载荷只展示测试内容，没有公开完整原始请求。

实验操作由本人和同学配合完成，使用 Codex 辅助抓包操作、提取校验与报告整理；结论以实际捕获数据为依据。课堂参考为《TCP/IP 协议概述与初识 Wireshark》，工具操作可参考 [Wireshark 用户指南](https://www.wireshark.org/docs/wsug_html_chunked/)。
