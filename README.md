# passwall-legacy-packages

给 PassWall 补回上游已放弃的协议包。只放配方，不放产物。

| 包 | 版本 | 说明 |
|---|---|---|
| `hysteria` | 1.3.5 | Hysteria **v1**，装到 `/usr/bin/hysteria`（动态链接 musl） |
| `shadowsocks-c` | 3.3.6 | shadowsocks-libev 的继任者，产出 `shadowsocks-libev-ss-{local,redir,server}` 三个子包，包名保持不变，PassWall 无需改动 |

---

## 加载方法

直接 clone 到 `package/` 下，不走 `src-git` feed。理由见文末「为什么不加进 feeds.conf」。

### 1. 先把默认 feeds 装好

```sh
./scripts/feeds update -a
./scripts/feeds install -a
```

hysteria 依赖 `feeds/packages` 里的 `lang/golang`，这一步不能跳。

### 2. 清掉同名包

`feeds install` 生成的 `package/feeds/<feed>/<pkg>` 是软链。要清的是三类目录：

| 目录名 | 来源 | 冲突原因 |
|---|---|---|
| `hysteria` | passwall-packages feed | 同名包 `hysteria`（v1 1.3.5） |
| `shadowsocks-libev` | passwall-packages feed（**旧版**） | 子包名 `shadowsocks-libev-ss-*` 重复 |
| `shadowsocks-c` | passwall-packages feed（**新版**） | 同上，子包名不变 |

**ss 的目录名取决于 feed 版本**：`openwrt-passwall-packages` 在提交 `e315f03` 里把 `shadowsocks-libev/` 删掉换成了 `shadowsocks-c/`。早于这个提交的树是 `shadowsocks-libev`，之后是 `shadowsocks-c`。

真正冲突的不是目录名，是**子包名**——我们的 `shadowsocks-c` 故意沿用 `shadowsocks-libev-ss-local` / `ss-redir` / `ss-server`，所以不管目录叫哪个，两边定义的是同一批包，`make` 会报重复。

先看一眼在哪：

```sh
find package/feeds -maxdepth 3 \
  \( -name hysteria -o -name shadowsocks-libev -o -name shadowsocks-c \) -print
```

确认无误后删掉：

```sh
find package/feeds -maxdepth 3 \
  \( -name hysteria -o -name shadowsocks-libev -o -name shadowsocks-c \) -print -delete
```

> 注意 `hysteria` 和 `hysteria2` 是两个不同的包（v1 / v2），`hysteria2` 不用动。
> `openwrt/packages` 官方 feed 在 24.10 里已经没有 `shadowsocks-libev` 了，所以通常只会命中 passwall 那一个。

### 3. clone 到 package/ 下

```sh
git clone https://github.com/M4TRIX04/passwall-legacy-packages.git \
  package/passwall-legacy-packages
```

确认落地：

```sh
ls package/passwall-legacy-packages/
# hysteria  shadowsocks-c
```

OpenWrt 的包扫描是递归的，`package/passwall-legacy-packages/hysteria/Makefile` 这种深度三层和 `package/feeds/xxx/` 一样能被扫到，不用额外配置。

### 4. 选中并编译

```sh
make menuconfig
```

- `Network` → `Web Servers/Proxies` → `shadowsocks-libev-ss-local` / `ss-redir` / `ss-server`
- `Network` → `hysteria`

```sh
make package/hysteria/compile V=s -j$(nproc)
make package/shadowsocks-c/compile V=s -j$(nproc)
# 或整包
make -j$(nproc)
```

产物在 `bin/packages/<arch>/base/`。

### 5. 更新配方

```sh
git -C package/passwall-legacy-packages pull
```

---

## 注意：重跑 feeds install 会把软链建回来

第 2 步的删除是**一次性**的。以后只要再跑 `./scripts/feeds install -a`（比如 `feeds update -a` 之后），`package/feeds/*/hysteria` 和 ss 那个目录会被重新创建，包名冲突再次出现。

重跑过 feeds 命令，就把第 2 步再执行一遍。可以直接存成脚本：

```sh
#!/bin/sh
find package/feeds -maxdepth 3 \
  \( -name hysteria -o -name shadowsocks-libev -o -name shadowsocks-c \) -print -delete
```

---

## 首次编译要联网下载 Go 工具链

hysteria 1.3.5 的 quic-go 是 `apernet/quic-go v0.34.1`，里面有：

```go
//go:build go1.21
var _ int = "The version of quic-go you're using can't be built on Go 1.21 yet. ..."
```

配方把工具链钉死在 1.20.14：

```make
GO_PKG_BUILD_VARS:=$(filter-out GOTOOLCHAIN=local,$(GO_PKG_BUILD_VARS)) GOTOOLCHAIN=go1.20.14
```

这**不是**配方里自带工具链。宿主 Go（24.10 是 1.23.x）会在编译时去 GOPROXY 下载 go1.20.14，构建日志里能看到：

```
go: downloading go1.20.14 (linux/amd64)
```

默认走 `proxy.golang.org`，国内环境大概率卡住。编之前先把代理设好：

```sh
export GOPROXY=https://goproxy.cn,direct
make package/hysteria/compile V=s -j$(nproc)
```

下载一次会缓存在 `dl/go-mod-cache`，后续编译不再需要网络。

---

## shadowsocks-c 为什么不用 v3.3.6 tag

该 tag 仍残留 libcork/libev 旧代码，而 libcork 是 git submodule，tarball 里不含，直接编会失败。所以固定到 master 上删掉遗留代码之后的 commit：

```
PKG_SOURCE_VERSION:=50f1ae54f737aa66f8edba05bdbf7250ff7f8a60
```

`PKG_MIRROR_HASH` 是 `skip`——git-archive 的 tarball 哈希随 git 版本变化，安全性由固定的 commit 保证。

构建走 CMake + `SS_DEPENDENCY_MODE=system`（复用 staging 里的 libuv/libcares/libsodium/mbedtls，不 bundle），并关掉 `SS_ENABLE_REGEX`（PassWall 不用 ACL 正则，可省掉 PCRE2 依赖）。实测 ss-local 从 924K（bundled + regex）降到 202K。

只产出 `local` / `redir` / `server` 三个子包，`ss-tunnel` 和 `ss-manager` 在 PassWall 里零引用，已删除。

---

## 验证环境

在 **OpenWrt 24.10.8 SDK，x86_64，GCC 13.3.0，musl** 下编过并上机跑通：`hysteria --version` 返回 `1.3.5`，`ss-local` / `ss-redir` 正常监听。配方是架构无关的。

---

## 为什么不加进 feeds.conf

`src-git` 也能用：

```sh
src-git passwall_legacy https://github.com/M4TRIX04/passwall-legacy-packages.git
```

但 `./scripts/feeds install -a -p passwall_legacy` 的 `-p` 只是「优先使用这个 feed」，不是隔离——遇到同名包时行为取决于安装顺序，而且 `feeds.conf` 存在时改 `feeds.conf.default` 根本不生效。直接 clone 到 `package/` 是确定性的，也方便 `git pull` 更新。

---

## 已知限制

- hysteria v1 已停止维护，协议与 v2 不互通，只在服务端仍是 v1 时有意义。
- `hysteria`（v1，装到 `/usr/bin/hysteria`）和 `hysteria2`（v2，装到 `/usr/bin/hysteria2`）是两个独立的包，包名和二进制名都不冲突，可以同时装，PassWall 里对应不同的节点类型。
- shadowsocks-c 锁的是 master commit 而非 tag，上游若 rebase / 强推需要重新确认 `PKG_SOURCE_VERSION`。
- 产物未签名。
