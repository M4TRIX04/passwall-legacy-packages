# passwall-legacy-packages

给 PassWall 补回上游已放弃的协议包。独立 feed，只放配方，不放产物。

| 包 | 版本 | 说明 |
|---|---|---|
| `hysteria` | 1.3.5 | Hysteria **v1**，静态二进制装到 `/usr/bin/hysteria` |
| `shadowsocks-c` | 3.3.6 | shadowsocks-libev 的继任者，产出 `shadowsocks-libev-ss-{local,redir,server}` 三个子包，包名保持不变，PassWall 无需改动 |

---

## 加载方法

### 1. 加进 feeds

在 OpenWrt 源码树（或 SDK）根目录编辑 `feeds.conf.default`，加一行：

```sh
src-git passwall_legacy https://github.com/M4TRIX04/passwall-legacy-packages.git
```

### 2. 更新并安装

```sh
./scripts/feeds update passwall_legacy
./scripts/feeds install -a -p passwall_legacy
```

`-p passwall_legacy` 只装这个 feed，不会把别的 feed 一起装进来。

### 3. 选中并编译

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

---

## 依赖说明

**hysteria 需要 `feeds/packages` 里的 `lang/golang`。** 配方 include 了 `$(TOPDIR)/feeds/packages/lang/golang/golang-package.mk`，所以默认 feeds 必须完整装过：

```sh
./scripts/feeds update -a
./scripts/feeds install -a
```

只装本 feed（`-a -p passwall_legacy`）会导致 hysteria 报 `No such file or directory: .../golang-package.mk`。

---

## 两个配方上的坑

**hysteria 1.3.5 编不过新版 Go。** 它把 quic-go 换成了 `apernet/quic-go v0.34.1`，那个版本里有：

```go
//go:build go1.21
var _ int = "The version of quic-go you're using can't be built on Go 1.21 yet. ..."
```

配方把工具链钉死在 1.20.14，宿主 Go 版本不影响：

```make
GO_PKG_BUILD_VARS:=$(filter-out GOTOOLCHAIN=local,$(GO_PKG_BUILD_VARS)) GOTOOLCHAIN=go1.20.14
```

**shadowsocks-c 不能用 v3.3.6 tag。** 该 tag 仍残留 libcork/libev 旧代码，而 libcork 是 git submodule，tarball 里不含，所以固定到 master 上删掉遗留代码之后的 commit `50f1ae54f737aa66f8edba05bdbf7250ff7f8a60`。`PKG_MIRROR_HASH` 是 `skip`——git-archive 的 tarball 哈希随 git 版本变化，安全性由固定的 `PKG_SOURCE_VERSION` 保证。

---

## 验证环境

这套配方在 **OpenWrt 24.10.8 SDK，x86_64，GCC 13.3.0，musl** 下编过并上机跑通：`hysteria --version` 返回 `1.3.5`，`ss-local` / `ss-redir` 正常监听。配方本身是架构无关的。

---

## 已知限制

- hysteria v1 已停止维护，协议与 v2 不互通，只在服务端仍是 v1 时有意义。
- `hysteria` 包名与 passwall feed 里的 hysteria v2 包**同名**，两者不能共存。
- shadowsocks-c 锁的是 master commit 而非 tag，上游若 rebase / 强推需要重新确认 `PKG_SOURCE_VERSION`。
