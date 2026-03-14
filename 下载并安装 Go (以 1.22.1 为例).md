#中转服务器DERP



（Designated Encrypted Relay for Packets）中转服务器，可以有效解决官方节点在海外延迟高或直连失败的问题。

由于您指定使用 **Go** 进行部署，以下是标准流程：

---

## 1. 环境准备

- **服务器**：一台拥有公网 IP 的 VPS（建议选择延迟低的区域，如香港或国内节点）。
- **域名**：DERP 强制要求 **HTTPS**（除非通过特殊参数跳过，但不推荐），需要一个指向该 VPS 的子域名。
- **开放端口**：
  - `443/TCP`：用于 HTTPS。
  - `3478/UDP`：用于 STUN 服务（帮助设备穿透 NAT）。

---

## 2. 安装 Go 环境

确保系统中已安装较新版本的 Go（建议 v1.20+）。

Bash

`# 下载并安装 Go (以 1.22.1 为例)
wget https://go.dev/dl/go1.22.1.linux-amd64.tar.gz sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.22.1.linux-amd64.tar.gz

# 配置环境变量

echo 'export PATH=$PATH:/usr/local/go/bin:$(go env GOPATH)/bin' >> ~/.bashrc
source ~/.bashrc`

---

## 3. 安装 derper 服务

直接使用 `go install` 拉取 Tailscale 官方提供的命令：

Bash

`# 安装最新版 derper go install tailscale.com/cmd/derper@main`

安装完成后，二进制文件会位于 `$(go env GOPATH)/bin/derper`。

---

## 4. 运行 DERP

最简单且推荐的运行方式是自动申请 SSL 证书（需确保 80 端口和 443 端口未被占用）：

Bash

`derper -hostname your.domain.com \\ -c=/root/derper.conf \\ -a :443 \\ -http-port 80 \\ -stun`

- `hostname`：填写你的域名。
- `c`：指定配置文件路径。
- `a`：指定 HTTPS 端口。
- `stun`：启用 STUN 服务。

> **小贴士**：如果你想在后台稳定运行，建议编写一个 **systemd** 服务文件。

---

## 5. 在 Tailscale 控制台配置

部署好服务端后，你需要告诉 Tailscale 客户端去连接它。

1. 登录 [Tailscale Admin Console](https://login.tailscale.com/admin/acls)。
2. 进入 **Access Control** (ACL)，在 JSON 配置中添加 `derpMap`：

JSON

`"derpMap": { "OmitDefaultRegions": false, "Regions": { "901": { "RegionID": 901, "RegionCode": "my-derp", "Nodes": [ { "Name": "1", "RegionID": 901, "HostName": "your.domain.com", "DERPPort": 443 } ] } } }`

- `901` 是自定义 ID（通常取 900 以上）。
- `OmitDefaultRegions` 若设为 `true`，则只使用你的自建节点。

---

## 6. 验证状态

在任意一台已安装 Tailscale 的设备上运行：

Bash

`tailscale netcheck`

如果你在输出的列表中看到了你的域名，并且延迟（Latency）非常低，说明中转成功！

---

### 进阶建议：安全性

默认情况下，任何知道你域名的人都可以使用你的 DERP 服务器。为了防止蹭网，建议在运行 `derper` 时开启 **验证模式**：

1. 在 DERP 服务器上也安装并启动 `tailscaled`。
2. 启动 `derper` 时增加参数 `-verify-clients`。

**需要我为您提供一个 systemd 自动化脚本，方便您一键启停服务吗？**
