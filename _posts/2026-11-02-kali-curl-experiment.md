---
layout: post
title: "Kali Linux 中 curl 的基本使用：网页获取、响应头分析与文件下载"
date: 2026-11-02
categories: [kali-linux, network-security]
---

## 一、实验目的

在网络安全学习中，除了浏览器，我们还经常需要直接在命令行中与网站服务器通信。`curl` 是一款常用的网络请求工具，可以获取网页内容、查看 HTTP 响应头、下载文件，也经常用于接口调试和网络故障排查。

本次实验使用 **Kali Linux** 中的 `curl`，完成以下三个操作：获取网页 HTML 源码、检查 HTTP 响应头、将网页保存为本地文件，并观察它们的输出差异。

## 二、实验环境

- **操作系统**：Kali Linux（VMware Workstation 虚拟机）
- **实验工具**：`curl`
- **测试地址**：<https://example.com>
- **实验日期**：2026 年 11 月 2 日

`example.com` 是用于文档和演示的示例域名，适合进行简单的 HTTP 请求练习。

## 三、实验操作与结果

### 1. 获取网页 HTML 源码

首先在 Kali 终端执行：

```bash
curl https://example.com
```

该命令向服务器发送 HTTP GET 请求，默认将响应正文直接输出到终端。实验中可以看到 `<!doctype html>`、`<title>Example Domain</title>` 等 HTML 标签，说明成功获取了网页源码。

由于返回的 HTML 没有专门进行排版，代码在终端里看起来比较紧凑，这不影响请求成功，也不是错误信息。

### 2. 查看 HTTP 响应头

接着执行：

```bash
curl -I https://example.com
```

`-I` 参数会发起 HEAD 请求，只显示响应头，不输出网页正文。本次实验观察到以下关键信息：

| 字段 | 本次实验结果 | 含义 |
| --- | --- | --- |
| HTTP 状态 | `HTTP/2 200` | 服务器成功处理请求 |
| `content-type` | `text/html; charset=utf-8` | 响应资源是采用 UTF-8 编码的 HTML |
| `server` | `cloudflare` | 响应中标识的服务器服务 |
| `allow` | `GET, HEAD` | 服务器声明该资源支持的请求方法 |
| `cf-cache-status` | `HIT` | 本次请求命中了 Cloudflare 缓存 |

通过这些字段，可以直观了解 HTTP 响应的状态、内容类型以及缓存情况。需要注意，响应头会随服务器配置和请求时间变化，并非每次都完全相同。

### 3. 下载网页到本地文件

最后执行：

```bash
curl -L https://example.com -o example.html
```

参数含义如下：

- `-L`：如果发生 HTTP 重定向，自动跟随新的地址。
- `-o example.html`：将响应内容保存为当前目录下的 `example.html` 文件，而不是直接打印在终端中。

截图中的传输进度到达 **100%**，本次共接收 **577 字节**，说明下载请求已完成。

下载后，还可以使用以下命令进行额外检查：

```bash
ls -lh example.html
cat example.html
```

第一条命令用于查看文件信息，第二条命令用于检查保存的 HTML 内容。这两条是后续验证命令，**不属于下方截图中已展示的操作**。

### 4. 实验截图

以下合并图片记录了上述三个命令在 Kali 终端中的真实执行情况：

![Kali Linux curl 三项实验终端截图]({{ '/assets/images/curl-experiment-2026-11-02.png' | relative_url }})

## 四、三个命令的区别

| 命令 | 核心作用 | 输出位置 |
| --- | --- | --- |
| `curl https://example.com` | 请求网页正文 | 终端 |
| `curl -I https://example.com` | 查看 HTTP 响应头 | 终端 |
| `curl -L https://example.com -o example.html` | 下载网页并在必要时跟随重定向 | 本地文件 |

## 五、实验总结

通过本次实验，我了解了 `curl` 的基本命令格式，能够在 Kali Linux 中使用它获取网页、查看服务器返回的 HTTP 状态和响应头，并将网页内容下载到本地。对比这三种操作后，我认识到 **网页正文、HTTP 响应头以及文件下载** 分别对应不同的使用需求。

对于网络安全学习而言，`curl` 不仅是一个下载工具，也可以帮助我们快速验证网站能否访问、观察服务端返回的信息，以及排查基本的 HTTP 通信问题。后续还可以继续学习请求参数、请求头、自定义 HTTP 方法及 API 测试等功能。
