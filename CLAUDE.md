# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

fscan 是一个用 Go 语言编写的内网综合扫描工具。支持主机存活探测、端口扫描、服务爆破、漏洞检测（包括 MS17-010、SMB Ghost）、Web 指纹识别，以及基于 YAML 格式的 Web 漏洞扫描（兼容 xray POC 格式）。

## 构建和开发命令

### 构建

```bash
# 标准编译
go build -ldflags="-s -w" -trimpath main.go

# 可选：使用 UPX 压缩体积
upx -9 fscan.exe

# 跨平台编译示例
# Windows 64位
GOOS=windows GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o fscan_windows_amd64.exe main.go

# Linux 64位
GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o fscan_linux_amd64 main.go

# macOS 64位
GOOS=darwin GOARCH=amd64 go build -ldflags="-s -w" -trimpath -o fscan_darwin_amd64 main.go

# macOS ARM64 (Apple Silicon)
GOOS=darwin GOARCH=arm64 go build -ldflags="-s -w" -trimpath -o fscan_darwin_arm64 main.go
```

### 编译参数说明

fscan 使用标准 Go 编译工具链，支持以下编译优化：

**ldflags 参数**：
- `-s`：去除符号表（strip symbol table），减小二进制文件大小
- `-w`：去除 DWARF 调试信息（strip debug info），进一步减小文件大小
- 组合使用 `-ldflags="-s -w"` 可显著减小可执行文件体积（约减少 30-40%）

**trimpath 参数**：
- `-trimpath`：从生成的可执行文件中删除所有文件系统路径
- 提高二进制文件的可重现性和安全性（不泄露本地路径信息）

**构建约束（Build Tags）**：
- fscan 代码库未使用构建标签（build tags），所有功能在所有平台上统一编译
- 跨平台兼容性通过 Go 标准库的平台抽象实现（如 `runtime.GOOS`、`runtime.GOARCH`）
- ICMP 功能在不同平台上使用 `golang.org/x/net/icmp` 包实现跨平台支持
- 某些功能在特定平台上有限制（如 ICMP 需要管理员权限）

**嵌入式资源（Embed）**：
- WebScan 模块使用 `//go:embed pocs` 将 380+ 个 YAML POC 文件嵌入到二进制文件中
- 编译时会自动打包 `WebScan/pocs/` 目录下的所有文件
- 无需额外的资源文件即可分发单个可执行文件

### 常用编译命令组合

```bash
# 开发调试版本（保留调试信息）
go build -o fscan main.go

# 生产发布版本（体积优化）
go build -ldflags="-s -w" -trimpath -o fscan main.go

# 交叉编译所有主流平台
for GOOS in darwin linux windows; do
  for GOARCH in 386 amd64 arm64; do
    # 跳过不支持的组合
    if [ "$GOOS" = "darwin" ] && [ "$GOARCH" = "386" ]; then continue; fi

    output_name="fscan_${GOOS}_${GOARCH}"
    if [ "$GOOS" = "windows" ]; then
      output_name="${output_name}.exe"
    fi

    echo "Building $output_name..."
    GOOS=$GOOS GOARCH=$GOARCH go build -ldflags="-s -w" -trimpath -o $output_name main.go
  done
done

# 编译后使用 UPX 进一步压缩（可选）
upx -9 fscan_*
```

### 依赖管理

```bash
# 下载依赖
go mod download

# 整理依赖（移除未使用的依赖）
go mod tidy

# 验证依赖
go mod verify

# 查看依赖树
go mod graph
```

### 运行

```bash
# 基本扫描
go run main.go -h 192.168.1.1/24

# 指定模块
go run main.go -h 192.168.1.1/24 -m ssh

# 指定端口
go run main.go -h 192.168.1.1/24 -p 22,80,443
```

### 测试

该代码库不包含单元测试。测试通过针对目标环境的手动执行完成。

## 架构总览

### 主要执行流程

1. **输入解析** (`common` 包)：解析命令行参数、IP 范围、端口范围和配置
2. **主机发现** (`Plugins/icmp.go`)：ICMP ping 扫描识别存活主机（可用 `-np` 跳过）
3. **端口扫描** (`Plugins/portscan.go`)：对存活主机进行 TCP 端口扫描
4. **服务检测** (`Plugins/*`)：针对特定协议的扫描和密码爆破
5. **Web 扫描** (`WebScan` 包)：指纹识别和基于 POC 的漏洞检测
6. **输出结果** (`common/log.go`)：结果输出到控制台，可选保存到文件

### 包结构

- **`main.go`**：程序入口，初始化 HostInfo，解析参数，调用 Plugins.Scan()

- **`common/`**：共享配置和工具函数
  - `flag.go`：命令行参数定义
  - `config.go`：默认字典、端口列表和全局状态（版本号、用户字典、密码字典、端口列表）
  - `ParseIP.go`：IP 范围解析（支持 CIDR、范围格式如 192.168.1.1-255、逗号分隔）
  - `ParsePort.go`：端口范围解析
  - `log.go`：带颜色输出的日志记录
  - `proxy.go`：HTTP/SOCKS5 代理配置

- **`Plugins/`**：服务特定的扫描模块
  - `scanner.go`：主调度器，将主机/端口路由到相应的插件
  - `portscan.go`：TCP 端口扫描器
  - `icmp.go`：ICMP ping 扫描用于主机发现
  - `base.go`：PluginList 映射表，注册端口号到扫描函数（如 "22" -> SshScan）
  - 服务扫描器：`ssh.go`、`smb.go`、`mysql.go`、`redis.go`、`ftp.go`、`mssql.go`、`postgres.go`、`oracle.go`、`rdp.go`、`mongodb.go`、`memcached.go`
  - 漏洞利用模块：`ms17010.go`、`ms17010-exp.go`、`CVE-2020-0796.go`（SMB Ghost）
  - `NetBIOS.go`：NetBIOS 枚举和域控识别
  - `webtitle.go`：HTTP 标题抓取和基础 Web 指纹识别
  - `fcgiscan.go`：FastCGI 协议扫描

- **`WebScan/`**：Web 漏洞扫描
  - `WebScan.go`：主入口，从嵌入式文件系统或自定义路径加载 POC
  - `InfoScan.go`：指纹检测
  - `lib/`：POC 引擎实现
    - `check.go`：指纹匹配和 POC 过滤
    - `eval.go`：CEL（通用表达式语言）表达式求值，用于 POC 规则
    - `client.go`：支持代理的 HTTP 客户端
    - `http.pb.go`：HTTP 请求/响应的 Protobuf 定义
  - `pocs/`：包含 380+ 个 YAML POC 文件的目录（使用 `//go:embed` 在编译时嵌入）

### 插件注册系统

插件在 `Plugins/base.go` 中通过 `PluginList` 映射表注册：

```go
var PluginList = map[string]interface{}{
    "22":      SshScan,
    "3306":    MysqlScan,
    "1000001": MS17010,     // 漏洞利用的特殊端口号
    "1000003": WebTitle,    // Web 扫描
    // ...
}
```

扫描器使用反射根据检测到的端口动态调用相应的函数。

### 并发模型

- 主协程池由 `-t` 参数控制（默认：600 个线程）
- `scanner.go` 中的 AddScan 函数使用基于 channel 的信号量限制并发扫描
- 每个服务扫描在自己的 goroutine 中运行
- WaitGroups 确保所有扫描在程序退出前完成
- Mutex 保护共享计数器（common.Num、common.End）

### 重要扫描模式（`-m` 参数）

- `all`（默认）：运行所有模块（ICMP → 端口扫描 → 服务扫描 → Web 扫描）
- `portscan`：仅执行端口扫描
- `icmp`：仅执行主机发现
- `webonly`：跳过端口扫描，直接探测常见 Web 端口的 HTTP/HTTPS
- `ssh`、`mysql`、`redis` 等：针对特定服务
- `ms17010`：扫描 MS17-010 漏洞

### 端口组

`common/config.go` 中的 `PortGroup` 映射定义了预设端口集：

- `main`：常用端口（21,22,80,81,135,139,443,445,1433,1521,3306,5432,6379,7001,8000,8080,8089,9000,9200,11211,27017）
- `db`：数据库端口
- `web`：Web 端口的扩展列表（80、8080、8443 等）
- `service`：常见服务端口

### Web POC 系统

- POC 是兼容 xray 格式的 YAML 文件
- 从嵌入式文件系统（`WebScan/pocs/`）或自定义路径（`-pocpath`）加载
- 使用 CEL 表达式进行匹配和提取
- 支持基于指纹的 POC 选择（仅根据检测到的 CMS/框架运行相关 POC）
- 通过 `-num` 参数限速（默认：20 个并发 POC 检查）

### WebScan 模块架构

WebScan 负责 Web 漏洞扫描，采用 CEL 表达式引擎和模块化设计。

#### 核心组件

- **WebScan.go**：主入口，从 `//go:embed pocs` 或 `-pocpath` 加载 POC，根据指纹过滤并并发执行
- **InfoScan.go**：指纹识别，包含 260+ 条规则（WAF、OA、中间件、CMS），支持 body/headers/cookie 正则匹配
- **lib/check.go**：POC 执行引擎
  - 普通模式：按顺序执行 rules
  - 爆破模式：使用 `sets` 字典生成笛卡尔积（如 Tomcat 弱口令、Shiro key）
  - 支持变量替换 `{{variable}}`、命名捕获组 `(?P<name>...)`、MD5 去重
- **lib/eval.go**：CEL 表达式引擎，30+ 自定义函数
  - 字符串：`bcontains`、`icontains`、`substr`、`startsWith`
  - 编码：`base64`、`urlencode`、`hexdecode`
  - 特殊：`shirokey`、`TDdate`（通达 OA）、`wait`（DNSLog）
- **lib/client.go**：HTTP 客户端，TLS 1.0+，支持 HTTP/SOCKS5 代理，连接池配置
- **info/rules.go**：指纹规则库和 POC 映射表（指纹 -> POC 别名）

#### 工作流程

1. 指纹识别：正则匹配 headers 和 body
2. POC 过滤：根据指纹查询 `PocDatas`，过滤相关 POC
3. 并发执行：worker 池（`-num` 控制）执行 POC，评估 CEL 表达式
4. 结果输出：`[+] PocScan <URL> <POC名称> [变量]`

#### POC YAML 格式

```yaml
name: poc-yaml-example
set:
  - key: var
    value: "value"
sets:  # 爆破字典（可选）
  - key: user
    value: ["admin", "root"]
rules:
  - method: POST
    path: /api/login
    body: '{"user":"{{user}}"}'
    search: '"token":"(?P<token>.*?)"'  # 命名捕获组
    expression: status == 200 && body.bcontains(b'success')
```

#### 特殊机制

- Shiro POC 默认测试 10 个 key（`-full` 测试 100 个），位置：lib/check.go:283
- DNSLog 支持使用 ceye.io，需 `-dns` 参数
- 基于指纹的智能过滤减少无效请求

## 常见开发模式

### 添加新的服务插件

1. 创建 `Plugins/newservice.go`，函数签名为：`func NewServiceScan(info *common.HostInfo)`
2. 在 `Plugins/base.go` 的 PluginList 中注册：`"PORT": NewServiceScan`
3. 在 `common/config.go` 的 PORTList 和 PortGroup 中添加默认端口
4. 如适用，在 `Userdict` 中添加默认凭据

### 添加新的 POC

在 `WebScan/pocs/` 目录中放置 YAML 文件。示例结构：

```yaml
name: poc-name
transport: http
rules:
  - method: GET
    path: /vulnerable/path
    expression: status == 200 && body.bcontains(b'marker')
```

### 错误处理

- 大多数函数使用 `defer recover()` 捕获 panic
- 通过 `common.LogError()` 和 `common.LogSuccess()` 记录错误
- 静默模式（`-silent`）抑制实时输出，用于自动化工具

## 关键全局变量

- `common.Threads`：并发扫描线程数
- `common.Timeout`：TCP 连接超时（默认：3秒）
- `common.WebTimeout`：HTTP 请求超时（默认：5秒）
- `common.NoPing`：跳过 ICMP 主机发现
- `common.NoPoc`：跳过 Web 漏洞扫描
- `common.IsBrute`：跳过密码爆破

## 值得注意的实现细节

- **IP 解析**：支持 /8 范围的智能采样（扫描每个 C 段的 .1 和 .254，加上随机 IP）
- **ICMP 实现**：使用原始套接字和自定义 ICMP 数据包构造（在 Linux/Windows 上需要 root/管理员权限）
- **密码爆破**：默认串行；使用 `-br N` 进行并发尝试（注意账户锁定）
- **MS17-010 利用**：内置 shellcode 生成用于添加用户（`-sc add`），但 README 推荐使用专用工具
- **代理支持**：HTTP 代理（`-proxy`）用于 Web 扫描；SOCKS5（`-socks5`）用于 TCP 连接（并非所有模块都支持）
- **哈希认证**：支持 SMB 的 pass-the-hash（`-hash`）和 WMI 执行

## 安全和授权上下文

此工具专为经授权的渗透测试和安全评估而设计。README 包含了全面的免责声明，要求在使用前获得合法授权。修改漏洞利用模块（ms17010-exp.go 等）时，请确保更改符合道德使用约束。
