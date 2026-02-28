# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

详细架构文档见 [docs/architecture.md](docs/architecture.md)。

## 项目概述

fscan 是一个用 Go 语言编写的内网综合扫描工具。支持主机存活探测、端口扫描、服务爆破、漏洞检测（包括 MS17-010、SMB Ghost）、Web 指纹识别，以及基于 YAML 格式的 Web 漏洞扫描（兼容 xray POC 格式）。

## 构建

```bash
# 开发调试版本（保留调试信息）
go build -o fscan main.go

# 生产发布版本（体积优化）
go build -ldflags="-s -w" -trimpath -o fscan main.go

# 跨平台编译
GOOS=windows GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o fscan_windows_amd64.exe main.go
GOOS=linux   GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o fscan_linux_amd64 main.go
GOOS=darwin  GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o fscan_darwin_amd64 main.go
GOOS=darwin  GOARCH=arm64 go build -ldflags="-s -w" -trimpath -o fscan_darwin_arm64 main.go
```

> **重要**：`WebScan/pocs/` 目录下的 POC YAML 文件通过 `//go:embed pocs` 在**编译期**打包进二进制。新增或修改 POC 后，必须重新编译才能生效。

```bash
# 运行示例
go run main.go -h 192.168.1.1/24
go run main.go -h 192.168.1.1/24 -m ssh
go run main.go -h 192.168.1.1/24 -p 22,80,443
```

## POC YAML 使用指南

POC 文件存放于 `WebScan/pocs/`，兼容 xray 格式。

**临时加载外部 POC（无需重编译）**：使用 `-pocpath <目录>` 从文件系统加载。

### 基础格式

```yaml
name: poc-name
transport: http
rules:
  - method: GET
    path: /vulnerable/path
    expression: response.status == 200 && response.body.bcontains(b'marker')
```

### 爆破字典（sets）

```yaml
name: poc-yaml-bruteforce
sets:
  - key: user
    value: ["admin", "root"]
  - key: pass
    value: ["admin", "123456"]
rules:
  - method: POST
    path: /login
    body: '{"user":"{{user}}","pass":"{{pass}}"}'
    expression: response.status == 200 && response.body.bcontains(b'success')
```

## 新增功能：search 字段扩展

原版 xray 格式的 `search` 字段仅用于提取命名捕获组并注入到后续 rule 的变量替换中。本项目在此基础上扩展：**所有命名捕获组的值会在 POC 命中时一并打印到输出**，适用于提取 accessToken、凭据、版本号等敏感信息。

**语法**：在任意 rule 中添加 `search` 字段，使用 Python 风格的命名捕获组 `(?P<name>...)`：

```yaml
name: poc-yaml-example-with-extraction
transport: http
rules:
  - method: POST
    path: /api/auth
    body: '{"username":"admin","password":"admin"}'
    search: '"accessToken":"(?P<accessToken>[^"]+)".*?"username":"(?P<username>[^"]+)"'
    expression: response.status == 200 && response.body.bcontains(b'accessToken')
```

命中时输出格式：

```text
[+] PocScan http://10.0.0.1:8848 poc-yaml-example-with-extraction
    * accessToken: eyJhbGc...
    * username: nacos
```

**说明**：

- `search` 匹配范围为 `响应头 + 响应体` 的拼接字符串
- 多个 rule 中的 `search` 均会被收集，后续 rule 提取的变量会覆盖同名变量
- 提取的变量同时注入 `variableMap`，可在后续 rule 中通过 `{{varName}}` 引用
- key 按字母序排列输出，空 key（未命名组）自动忽略
- 实现位置：`WebScan/lib/check.go`，函数 `executePoc` 和 `CheckMultiPoc`

## 新增功能：-hs 参数（POC 请求响应保存）

当 POC 命中时，可使用 `-hs <目录>` 参数保存触发漏洞的 HTTP 请求和响应：

```bash
fscan -hs ./poc-output -u http://target.com -pocname weblogic
```

命中后会在指定目录生成 `{poc_name}_{url_hash}_{timestamp}.txt` 文件，包含完整的请求头/请求体和响应头/响应体。

## 安全和授权上下文

此工具专为经授权的渗透测试和安全评估而设计，使用前须获得合法授权。
