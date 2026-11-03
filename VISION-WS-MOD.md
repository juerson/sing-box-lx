# WS 传输下的 VLESS Vision 流控 —— sing-box-lx 客户端魔改说明

> 目的：让 **sing-box-lx** 在 **WebSocket** 传输上也能使用 `xtls-rprx-vision` 流控
> （与 CF Workers「流控版」服务端配套；Worker 代码在另一个仓库）。
> 最后更新：2026-11-04

## 0. 基线与集成方式（重要）

| 组件 | 基线 | 说明 |
|---|---|---|
| **sing-box-lx** | `v1.14.4-lx.30`（`upstream.version` = `1.14.4`） | 工作区 `git describe` = `v1.14.4-lx.30-5-ga3bb4a04`（lx.30 之后 5 个文档提交） |
| **上游 sing-box** | 挂在 `1.14.4` 线上 | 版本号由 ldflags 注入，源码里 `constant/version.go` 是 `"unknown"` |
| **sing-vmess** | 上游 `v0.2.9-0.20261019223527-a0c58542bd9d` | 与 `go.mod:65` 的 require 行一致；整包拷进 `./sing-vmess/`（嵌套 `.git` 已删除） |

### 代码来源

`./sing-vmess/` 是 [SagerNet/sing-vmess](https://github.com/SagerNet/sing-vmess) **dev 分支**
的完整副本，基线为 `go.mod:65` 锁定的 `v0.2.9-0.20261019223527-a0c58542bd9d`
（pseudo-version 中的 commit 即 `a0c58542bd9d`），在此基础上做了本文第 2 节的本地修改。
除 `vless/vision.go` 外，其余文件保持上游原样。

上游许可证为 GPL-3.0（`./sing-vmess/LICENSE` 原样保留），下游分发由此构建的二进制时
同样受 GPL-3.0 约束。日后同步上游时，以该 pin 为差异起点（对比上游 pin 与本地副本），
不要直接删除 `replace` 后重新 `go get`。

集成方式：

- `sing-vmess` 以**本地目录**形式放进仓库，`go.mod` 末尾用
  `replace github.com/sagernet/sing-vmess => ./sing-vmess` 指向它；
- **module path 保持 `github.com/sagernet/sing-vmess` 不变**，因此
  `protocol/vless/encryption/lx_vision.go` 里 `//go:linkname .../vless.tlsRegistry`
  （SPEC 105）仍然解析到本地副本；
- sing-box 本体的 Go 代码不因本魔改改动（`protocol/vless/` 仍只是调用方）。

> 日后若要重拉源码魔改，请以**上述 pin 为基线**；若上游已更新，需重新核对
> `sing-vmess/vless/vision.go` 的四处改动是否仍适用。

## 1. 为什么要改

sing-box 的 vless 由 `github.com/sagernet/sing-vmess` 提供。它的 vision 实现
（`sing-vmess/vless/vision.go`）**假设外层是裸 TLS 连接**：

1. `NewVisionConn` 要求底层能解出 `*crypto/tls.Conn`，否则返回
   `vision: not a valid supported TLS connection`；
2. `Write` 在识别到 **TLS 1.3** 后会把 `directWrite` 置真，进入 **XTLS direct（splice）**，
   把后续字节**直接写底层 netConn**。

在 **WebSocket** 传输下：① 外层不是 `*tls.Conn`；② 没有可 splice 的裸连接。
于是 ws + vision **要么建连即被拒、要么一进入应用数据就错乱断开**
（典型现象：测速通、一用就断）。

sing-box-lx 里 Vision 原本已能跑在 TCP+TLS、REALITY 和 VLESS-encryption
（SPEC 105）上，缺的只是 WS 这一条路径。因此对本地 `sing-vmess` 做最小魔改，
使其在 WS 下**退化为「纯 padding、不 splice」**模式。

## 2. 改了哪些文件

### 2.1 `sing-vmess/vless/vision.go`（核心四处）

**(1) `NewVisionConn`：非裸 TLS 时不再报错**
- 删除 `if !loaded { return nil, E.New("vision: not a valid supported TLS connection...") }`；
- 仅当 `loaded == true` 时才反射读取底层 TLS 的 `input` / `rawInput`，否则保持 `nil`。

**(2) `Write`：禁用 XTLS direct**
- 删除 `if c.enableXTLS { c.directWrite.Store(true); specIndex = i; command = commandPaddingDirect }` 分支；
- 一律使用 `commandPaddingEnd`；
- 删除因切换 splice 而不再需要的 `var specIndex int`。

**(3) `Read`：`commandPaddingDirect` 分支不做 splice**
- 去掉 `c.directRead = true` 与读取 `c.input` / `c.rawInput` 的代码，改为按「结束填充」处理
  （`withinPaddingBuffers = false`，重置 remaining）。

**(4) import 清理**
- 因上述改动不再使用，已从 import 中删除 `io`、`time`。

> 保留的死路径（不影响行为，便于日后对照上游）：
> `directWrite` / `directRead` 字段及其读取分支仍在，但**永远不会被置真**；
> `filterTLS` 仍会判定 TLS 1.3 并设置 `enableXTLS`，该值只用于 Trace 日志。
>
> 其余逻辑（padding 长度算法、uuid 首块、`filterTLS`、Continue/End 状态机）**保持原样**。

### 2.2 `go.mod`

追加一行，让 sing-box-lx 使用本目录里的 sing-vmess（而不是下载官方版）：

```
replace github.com/sagernet/sing-vmess => ./sing-vmess
```

当前快照（未带 lx 标记）：

```
// lx:end utls-firefox148
// 新增
replace github.com/sagernet/sing-vmess => ./sing-vmess
```

按 CONSTITUTION 对上游文件的最小改法，这段应放进 `// lx:` 标注区（紧跟现有四个
本地 replace 之后，例如 `// lx:begin vision-ws` … `// lx:end vision-ws`），
合并 `upstream/stable` 时冲突面更小。

**sing-box 自身的 Go 代码无需改动。**

### 2.3 与 SPEC 105（Vision over VLESS Encryption）的关系

- 本地副本的 `vless.tlsRegistry` 仍是唯一入口，`encryption/lx_vision.go` 的
  `//go:linkname` 注册继续生效（`replace` 只换源码位置，不改包路径）；
- SPEC 105 的条目让 `*CommonConn` 被当作 TLS conn（`loaded=true`），继续走
  reflect 读 `input` / `rawInput`；
- WS 场景下底层是 WS conn，注册表没有匹配项 → `loaded=false` → 走本魔改的
  「纯 padding」路径。两套改动互不冲突。

## 3. 目录结构（sing-box-lx 自包含）

```
sing-box-lx/
├─ go.mod                          # 含 replace => ./sing-vmess
├─ upstream.version                # 上游基线 1.14.4
├─ Makefile.lx                     # LX_TAGS / LX_LDFLAGS / lx-build
├─ sing-vmess/                     # vendored + 魔改（无 .git）
│  ├─ go.mod
│  └─ vless/
│     ├─ vision.go                 # ★ 魔改点
│     ├─ vision_utls.go
│     ├─ client.go
│     └─ ...
├─ protocol/vless/                 # 调用方
│  └─ encryption/lx_vision.go      # SPEC 105：注册 CommonConn
└─ ...
```

## 4. 编译

标准做法是走仓库的 `Makefile.lx`，**不要手搓 tag 清单**：

```bash
make -f Makefile.lx lx-version     # 打印 1.14.4-lx.N
make -f Makefile.lx lx-build       # 生成 ./sing-box
```

要点：

- `LX_TAGS` 是唯一权威的 tag 集合（`Makefile.lx:50`），已含 `with_clash_api`、
  `with_gvisor`、`with_quic`、`with_utls`、`with_wireguard`、`with_naive_outbound`、
  `with_purego`、`badlinkname`、`with_xhttp`、`with_awg` 等；
- 版本号经 `LX_LDFLAGS` 注入，`constant/version.go` 保持上游原样；
- 必须带 `-checklinkname=0`（`LX_LDFLAGS` 已包含），否则 `badlinkname` /
  `tfogo_checklinkname0` 的 `go:linkname` 会被 Go 1.24+ 链接器拒绝；
- 入口是 `./cmd/sing-box`（**不要**用 `go build .`）。

Windows 下没有 make 时的等价手工命令：

```bat
set CGO_ENABLED=0
go build -trimpath -tags "%LX_TAGS%" -ldflags "-X github.com/sagernet/sing-box/constant.Version=1.14.4-lx.N -checklinkname=0 -s -w -buildid=" -o sing-box.exe ./cmd/sing-box
```

> 旧文档里「不要加 `with_naive_outbound`，否则会牵出 cronet-go 报错」的提示
> **在 sing-box-lx 已不适用**：`LX_TAGS` 同时带 `with_naive_outbound` 与
> `with_purego`，靠 cronet 预编译产物在 `CGO_ENABLED=0` 下构建。

## 5. 配套要求（服务端）

服务端（CF Workers「流控版」）必须**同样只发 `End`、不发 `Direct`**，否则对端仍会
进入无法实现的 direct 模式。该 Worker 代码在本仓库之外（`src/fctrl/traffic.js`
不在 sing-box-lx 中），这里只作为协议约束记录。

## 6. 上游代价与注意

- 该魔改**只为让 vision 能跑在 WS 上**，**没有任何 splice 性能收益**
  （CF Workers + WS 本就无法直通）。
- 改动是**全局生效**的：TCP+TLS 场景也会从 XTLS direct 退化为纯 padding，
  功能不受影响但失去 splice 性能。如果只想在 WS 上退化，应改成 build-tag 或
  按 transport 判定，而不是无条件禁用。
- 因为是**整包 vendoring**，上游 `sing-vmess` 的后续提交不会随 `go.mod` 自动跟进；
  合并 `upstream/stable` 遇到 sing-vmess 版本 bump 时，需要把差异手工合进
  `./sing-vmess/`，不能直接删掉 `replace`。
- 若要客户端走 **TCP + TLS**（真正需要 direct 的场景），应使用**原版** sing-vmess。
- 合并/发布前建议跑 `make -f Makefile.lx lx-check`，用样例配置校验新二进制。