# 自建 VPN 完整教學：用 wg-easy 架設 WireGuard 伺服器

從只有 SSH 可以登入的 Ubuntu A1 主機開始，完成 Docker 安裝、Oracle 網路設定、免費 DuckDNS 子網域、Caddy HTTPS 反向代理、wg-easy 初始化與手機 VPN 連線測試。完成後，直接用瀏覽器開啟 HTTPS 網址管理 VPN。

本文以 **2026-11-11** 查閱的 **wg-easy v15.5.0 專案範例**為基準，部署路徑統一使用 **`/opt/wg-easy`**。本文所稱「官方範例」是指 **wg-easy 專案自己發布的範例**；wg-easy 是第三方專案，並非 WireGuard 官方推出的管理介面。

> 適用範圍：全新的 Oracle Cloud A1、Ubuntu 24.04 LTS、ARM64、已有公網 IPv4，且可以用 `ubuntu` 帳號與 SSH 私鑰登入。本文先驗證 IPv4 VPN，不涵蓋完整的 IPv6 對外連線設定。

## 目錄

1. [部署方式與準備事項](#1-部署方式與準備事項)
2. [確認主機環境](#2-確認主機環境)
3. [安裝 Docker Engine 與 Compose](#3-安裝-docker-engine-與-compose)
4. [Oracle 網路規則](#4-oracle-網路規則)
5. [建立 Compose 設定](#5-建立-compose-設定)
6. [啟動與檢查服務](#6-啟動與檢查服務)
7. [設定免費網域與 Caddy HTTPS 反向代理](#7-設定免費網域與-caddy-https-反向代理)
8. [wg-easy 初始化](#8-wg-easy-初始化)
9. [建立裝置與驗證 VPN](#9-建立裝置與驗證-vpn)
10. [日常管理與更新](#10-日常管理與更新)
11. [日後調整](#11-日後調整)
12. [參考文件](#12-參考文件)

## 1. 部署方式與準備事項

### 本文採用的配置

| 項目 | 配置 |
|---|---|
| 主機 | 現有 Oracle A1，Ubuntu 24.04 ARM64 |
| 部署目錄 | `/opt/wg-easy` |
| 設定檔 | `/opt/wg-easy/compose.yaml` |
| 容器映像 | `ghcr.io/wg-easy/wg-easy:15`，沿用專案範例標籤 |
| VPN 入口 | A1 公網 IPv4 的 UDP `51820` |
| 管理介面 | `https://vpn.myserver.duckdns.org`，由 Caddy 反向代理 |
| Caddy 部署目錄 | `/opt/caddy`，包含 `compose.yaml` 與 `Caddyfile` |
| 反向代理網路 | 獨立的 Docker bridge 網路 `caddy`，供 Caddy 與後端服務共用 |
| HTTPS 入口 | TCP `443`；TCP `80` 用於憑證驗證及轉址 |
| Docker 內部網路 | 沿用範例 `10.42.42.0/24` 與 IPv6 ULA 網段 |
| wg-easy 容器 IPv4 | 沿用範例 `10.42.42.42` |
| VPN 測試範圍 | IPv4；本文設定 `DISABLE_IPV6=true` |
| 持久資料 | 沿用範例的 `etc_wireguard` named volume |

`myserver` 是本文的示範名稱，請換成自己在 DuckDNS 申請的名稱。

這裡有三種不同的位址：

- **A1 公網 IP**：手機／電腦建立 VPN 連線時使用。
- **Docker 容器 IP**，例如 `10.42.42.42`：Docker 內部使用，不是手機連線時填寫的伺服器地址。
- **VPN 裝置 IP**：裝置連上 VPN 後使用，由 wg-easy 設定分配。

管理頁網域由 DNS 指向 A1 公網 IP，再由 Caddy 依網址轉發到 wg-easy。HTTPS 管理頁與 UDP `51820` 的 VPN 連線是兩個入口。

### 需要先準備

- A1 的公網 IPv4。
- 能登入 Oracle Console、修改主機所屬網路規則的權限。
- SSH 私鑰，以及能登入 A1 的 SSH 客戶端。
- 一台可安裝 WireGuard App 的手機，用行動網路測試。
- 可登入 DuckDNS 的帳號；已有自己的網域也可以使用。
- A1 的 TCP `80`、`443` 尚未被其他服務占用。

**Linux 指令**在 A1 的 SSH 終端機執行；標為 **PowerShell** 的指令在自己的 Windows 電腦執行。指令中的示範路徑、IP 或檔名請依註解替換。

### 成本範圍

本文在現有 A1 上執行容器。費用仍取決於該帳號的 Compute、磁碟、對外流量等用量與免費額度，並不因為使用 Docker 或 wg-easy 就自動變成全部免費。

查閱日的 OCI 價目表列出日本區域每月前 **10 TB 對外傳輸免費**，傳入流量免費。VPN 會轉送實際資料，背景下載、備份和遠端串流也可能計入這些流量。部署前後可在 OCI 的 Cost Analysis 與用量資訊確認自己的帳號狀況。

來源：[OCI 價目表](https://www.oracle.com/cloud/price-list/)、[Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)。

## 2. 確認主機環境

在 A1 執行：

```bash
cat /etc/os-release
uname -m
```

預期：

- Ubuntu `24.04`，代號 `noble`。
- 架構 `aarch64`；Docker 套件架構名稱則是 `arm64`。

若不是此環境，先依照 [Docker 安裝文件](https://docs.docker.com/engine/install/ubuntu/)調整套件來源。本文後續命令明確使用 Ubuntu Noble 與 ARM64。

## 3. 安裝 Docker Engine 與 Compose

本節使用 Docker 自己維護的 apt 套件來源。以下假設是沒有其他 Docker／Podman 部署的乾淨主機。

### 3.1 安裝必要工具與簽署金鑰

```bash
sudo apt update
sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### 3.2 新增 Docker 套件來源

```bash
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<'EOF'
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: arm64
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

### 3.3 安裝並啟動

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

### 3.4 驗證安裝

```bash
sudo docker run --rm hello-world
sudo docker compose version
```

看到 `Hello from Docker!` 與 Compose 版本，代表基本環境可用。

**本教學需要 Docker Compose 2.36.0 以上**，用來固定 wg-easy 在兩張網路中的介面名稱與預設閘道。若版本太舊，先更新 Compose plugin，再繼續：

```bash
sudo apt update
sudo apt install --only-upgrade docker-compose-plugin
sudo docker compose version
```

本文統一使用 `sudo docker ...`，因此不需要額外把帳號加入 `docker` 群組。

接著確認 WireGuard 核心模組能載入：

```bash
sudo modprobe wireguard
```

沒有輸出通常代表成功。若出現 `Module wireguard not found` 或其他錯誤，先記下 `uname -r` 和完整錯誤，再處理目前核心與模組的相容性，不要直接跳過。

來源：[Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)、[LinuxServer WireGuard 的核心模組說明](https://docs.linuxserver.io/images/docker-wireguard/)。

## 4. Oracle 網路規則

### 4.1 確認原本網路已能使用

主機應已有公網 IPv4，且可以正常 SSH 登入；所屬公用子網路應有通往 Internet Gateway 的路由。

沿用既有 VCN 與子網路，不需要為此再建立一套網路。

### 4.2 新增 VPN 與 HTTPS 的 ingress rules

在 Oracle Console 找到 A1 **實際使用的子網路及關聯的 Security List**。常見路徑：

**Networking → Virtual cloud networks → 你的 VCN → Subnets → 使用中的公用子網路 → Security lists → 使用中的 Security List → Add ingress rules**。

新增以下三條 **Ingress Rules**：

| 欄位 | VPN | HTTP／憑證驗證 | HTTPS 管理頁 |
|---|---|---|---|
| Stateless | 不勾選 | 不勾選 | 不勾選 |
| Source Type | CIDR | CIDR | CIDR |
| Source CIDR | `0.0.0.0/0` | `0.0.0.0/0` | `0.0.0.0/0` |
| IP Protocol | UDP | TCP | TCP |
| Source Port Range | All／留空 | All／留空 | All／留空 |
| Destination Port Range | `51820` | `80` | `443` |
| Description | `WireGuard VPN` | `HTTP for Caddy` | `HTTPS for Caddy` |

**連接埠填在 Destination Port Range，不是 Source Port Range。** 容器顯示 `Up` 只代表程式在主機上運作，並不代表 Oracle 已允許外部連線。

保留原有 SSH 與其他必要規則。本文假設沿用允許一般對外流量的 egress 設定；若有另外限制 egress 或主機防火牆，也需要確認 VPN、DNS 查詢及憑證申請所需流量能通過。

Security List 規則會套用至與它關聯的子網路；也可以依自己的管理方式，改用掛在該 A1 VNIC 上的 NSG。

來源：[Oracle Security Lists](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securitylists.htm)、[wg-easy Basic Installation](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/basic-installation/)、[Caddy Automatic HTTPS](https://caddyserver.com/docs/automatic-https)。

## 5. 建立 Compose 設定

### 5.1 建立獨立的 Caddy 網路與部署目錄

先建立共用的反向代理網路；若已存在，沿用它：

```bash
sudo docker network inspect caddy >/dev/null 2>&1 || sudo docker network create caddy
sudo mkdir -p /opt/wg-easy
cd /opt/wg-easy
```

兩份 Compose 都把 `caddy` 宣告為 external network，這樣可以先啟動 wg-easy，稍後再設定 Caddy。與官方範例「由 Caddy Compose 建立同名網路」的差別是建立方式；服務仍透過獨立的 `caddy` 網路互通。日後單獨移除其中一個 Compose 專案，也不會由它自動刪掉這張共用網路。

### 5.2 寫入設定

以下保留 wg-easy v15.5.0 原始範例的網路、固定容器 IP 與主要容器選項。整段貼上，包含最後的 `EOF`；結束標記 `EOF` 必須單獨一行。

```bash
sudo tee /opt/wg-easy/compose.yaml >/dev/null <<'EOF'
volumes:
  etc_wireguard:

services:
  wg-easy:
    environment:
      - DISABLE_IPV6=true

    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    networks:
      wg:
        ipv4_address: 10.42.42.42
        ipv6_address: fdcc:ad94:bacf:61a3::2a
        interface_name: eth0
        gw_priority: 1
      caddy:
        interface_name: eth1
    volumes:
      - etc_wireguard:/etc/wireguard
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv6.conf.all.forwarding=1
      - net.ipv6.conf.default.forwarding=1

networks:
  wg:
    driver: bridge
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: 10.42.42.0/24
        - subnet: fdcc:ad94:bacf:61a3::/64
  caddy:
    external: true
    name: caddy
EOF
```

### 5.3 和原始範例的差異

| 項目 | 原始範例 | 本文 | 原因 |
|---|---|---|---|
| 管理頁映射 | `51821:51821/tcp` | 移除這條映射 | 只讓 Caddy 從 Docker 內部網路連入 |
| VPN IPv6 | 預設啟用 | `DISABLE_IPV6=true` | 本教學先驗證 IPv4 VPN |
| 反向代理網路 | 基本範例只有 `wg` 網路 | 額外加入獨立的 `caddy` 網路 | 讓 Caddy 與多個後端服務共用反向代理網路 |
| 出口介面及閘道 | 單一網路 | `wg` 固定為 `eth0` 並設定 `gw_priority: 1`；`caddy` 為 `eth1` | 新增第二張網路後，仍讓 VPN 對外流量走原本的 `wg` 網路 |

`interface_name` 固定容器介面名稱；`gw_priority` 選擇預設閘道。不要用 `priority` 代替，Docker 文件明確指出它不負責固定介面名稱或選擇預設閘道。這裡需要 Compose 2.36.0 以上。

其他核心配置保留原樣。Docker 內部 IPv6 網路仍沿用範例，但 `DISABLE_IPV6` 讓 wg-easy 不替 VPN 介面與裝置啟用 IPv6；兩者是不同層的網路設定。依專案文件，即使管理頁仍顯示 IPv6 欄位，也不代表它已在 VPN 中使用。

**這不會關閉手機或電腦原有的 IPv6。** 本文只能確認 IPv4 是否經 A1 轉送；若要求 IPv6 也經 VPN，需要另行完成並驗證整條 IPv6 路徑。

來源：[v15.5.0 原始 Compose](https://github.com/wg-easy/wg-easy/blob/v15.5.0/docker-compose.yml)、[Optional Configuration](https://wg-easy.github.io/wg-easy/latest/advanced/config/optional-config/)、[Docker port publishing](https://docs.docker.com/engine/network/port-publishing/)、[Docker volumes](https://docs.docker.com/engine/storage/volumes/)、[Compose 網路介面與閘道設定](https://docs.docker.com/reference/compose-file/services/#networks)。

## 6. 啟動與檢查服務

```bash
cd /opt/wg-easy
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose up -d
```

`config --quiet` 沒有輸出代表設定格式檢查通過。若有錯誤，先修正再繼續。

查看狀態與紀錄：

```bash
sudo docker compose ps
sudo docker compose logs --tail=50 wg-easy
```

預期容器保持 `Up`／`running`，可能另顯示健康檢查狀態。剛啟動時可以稍等片刻，再檢查一次。

此時還沒有 HTTPS 管理入口，先完成下一節的 DuckDNS 與 Caddy 設定，再開啟管理頁。本文沒有把 `51821` 映射到主機，因此不使用 `http://127.0.0.1:51821` 測試。若容器持續重新啟動或有錯誤，先看紀錄。

> Docker 的 bridge 網路會建立轉送、連接埠映射與 NAT 規則。本文沿用 Docker 預設網路管理方式，沒有清空 Oracle 映像的防火牆，也沒有停用 Docker 的防火牆管理。不要把「清空所有 iptables」當成通用解法。

來源：[Docker packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/)。

## 7. 設定免費網域與 Caddy HTTPS 反向代理

本節以 `vpn.myserver.duckdns.org` 作為管理網址。**所有 `myserver` 都要換成自己申請的名稱**；如果使用自己的網域，則換成準備好的完整主機名稱。

### 7.1 申請 DuckDNS 名稱並指向 A1

1. 開啟 [DuckDNS](https://www.duckdns.org/)，使用網站提供的方式登入。
2. 在 Domains 輸入通用名稱，例如 `myserver`，按 **add domain**。
3. 將 **current ip** 改成 **A1 的公網 IPv4**，按 **update ip**。不要保留成自己電腦目前上網的出口 IP。
4. 本文先使用 IPv4；沒有正確設定 A1 的 IPv6 時，不要填入 IPv6 地址。

DuckDNS 提供的是 `myserver.duckdns.org` 這種免費子網域。它的下一層子網域會自動解析到相同 IP，因此可使用 `vpn.myserver.duckdns.org`，日後也能用 `files.myserver.duckdns.org` 區分服務，不必在 DuckDNS 逐個新增。不同服務由 Caddy 按網址轉發。

若 A1 的公網 IP 日後改變，記得更新 DuckDNS。這份教學先手動設定 IP，沒有另外安裝 DDNS 自動更新程式。

來源：[DuckDNS About](https://www.duckdns.org/about.jsp)、[DuckDNS API](https://www.duckdns.org/spec.jsp)、[LinuxServer 關於 DuckDNS 下一層子網域的說明](https://docs.linuxserver.io/general/swag/#nextcloud-subdomain-reverse-proxy-example)。

### 7.2 確認 DNS 和連接埠

在自己的 Windows PowerShell 執行，換成自己的完整網域：

```powershell
Resolve-DnsName vpn.myserver.duckdns.org -Type A
Resolve-DnsName vpn.myserver.duckdns.org -Type AAAA
```

- A 記錄應回傳 **A1 的公網 IPv4**。
- AAAA 沒有記錄可以；若有記錄，必須確實指向已設定好網路與防火牆的 A1 IPv6。錯誤的 AAAA 記錄也可能影響驗證或存取。

在 A1 確認 `80`、`443` 是否已有程式監聽：

```bash
sudo ss -ltnp '( sport = :80 or sport = :443 )'
```

乾淨主機通常只會顯示表頭。若已經有 Caddy、Nginx 或其他服務使用這兩個連接埠，先確認現有部署，不要再啟動第二套服務搶用相同埠。

### 7.3 確認獨立的 Caddy 網路

第 5 節已建立共用網路 `caddy`，並讓 wg-easy 加入。在 A1 確認：

```bash
sudo docker network inspect caddy --format '{{range .Containers}}{{println .Name}}{{end}}'
```

此時應能看到 `wg-easy`；下一步啟動 Caddy 後，兩個容器都會出現在這張網路中。

Caddy 只加入 `caddy` 網路；wg-easy 同時保留原本的 `wg` 網路與新增的 `caddy` 網路。日後其他網頁服務也可以加入 `caddy`，不必共用 VPN 原本的 `wg` 網路。

### 7.4 建立 Caddy 的兩個設定檔

在 A1 執行：

```bash
sudo mkdir -p /opt/caddy
cd /opt/caddy
```

**第一個檔案：`compose.yaml`。這段可直接貼上。**

```bash
sudo tee /opt/caddy/compose.yaml >/dev/null <<'EOF'
services:
  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80/tcp"
      - "443:443/tcp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - caddy

networks:
  caddy:
    external: true
    name: caddy

volumes:
  caddy_data:
  caddy_config:
EOF
```

**第二個檔案：`Caddyfile`。先把完整網域換成自己的名稱，再貼上。**

```bash
sudo tee /opt/caddy/Caddyfile >/dev/null <<'EOF'
vpn.myserver.duckdns.org {
    reverse_proxy wg-easy:51821
}
EOF
```

Caddy 根據這個公開網域申請憑證，再把管理頁請求轉發到 `wg-easy:51821`。

這個配置與 wg-easy 專案的 Caddy 範例使用相同的反向代理原理，配合本教學調整如下：

| 項目 | 本文設定 |
|---|---|
| Caddy 映像 | 官方 `caddy:2-alpine`，追蹤 Caddy 2 Alpine 標籤 |
| 共用 Docker 網路 | 獨立的 `caddy` 網路，預先建立，兩個 Compose 專案都以 external 方式加入 |
| 後端管理埠 | 沿用 wg-easy 預設的 `51821`，不另改成 `80` |
| 網域設定 | 直接填在 Caddyfile |
| 公開憑證 | 不加入 `tls internal`，由 Caddy 自動申請公開憑證 |
| HTTP/3 | 本文只映射 TCP `443`，先以 HTTP/1.1、HTTP/2 使用；不需要 UDP `443` |

**不要照抄 `tls internal`**：那是 Caddy 內部 CA 的憑證，需要用戶端另外信任它；本教學要的是一般瀏覽器可驗證的公開憑證。

`caddy_data` 保存憑證和 ACME 帳號資料，`caddy_config` 保存 Caddy 設定狀態。保留這些 volume，避免每次重建都重新註冊或申請憑證。

來源：[wg-easy Caddy 範例](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/caddy/)、[Caddy 官方容器映像](https://hub.docker.com/_/caddy)、[Caddy Automatic HTTPS](https://caddyserver.com/docs/automatic-https)、[Compose 共用既有網路](https://docs.docker.com/compose/how-tos/networking/)。

### 7.5 驗證設定並啟動

確認 DNS 正確、第 4 節的 ingress rules 已開放，再執行：

```bash
cd /opt/caddy
sudo docker compose config --quiet
sudo docker compose pull
sudo docker compose run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

確認沒有設定錯誤後啟動：

```bash
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 caddy
```

Caddy 會自動申請憑證、後續自動續期，並將一般 HTTP 請求轉到 HTTPS。第一次申請可能需要稍等片刻。本配置使用從外部連入的 ACME 驗證方式，不需要 DuckDNS token。

### 7.6 開啟 HTTPS 管理頁

在電腦或手機的瀏覽器開啟自己的網址：

```text
https://vpn.myserver.duckdns.org
```

正常應顯示 wg-easy 初始化或登入頁，而且瀏覽器沒有憑證警告。

這裡的 HTTPS 保護的是**管理頁**。WireGuard 的 VPN 連線仍走 UDP `51820`，不需要因為管理網址改變就重新匯出所有裝置設定。

### 7.7 常見問題

**憑證申請顯示 `Timeout during connect (likely firewall problem)`**

先看錯誤中的 IP 是否等於 A1 公網 IP，再確認 **實際關聯的 Security List／NSG** 已新增 TCP `80`、`443` 的 ingress rules。只開 VPN 的 UDP `51820` 不夠。

在 Windows PowerShell 測試：

```powershell
Test-NetConnection vpn.myserver.duckdns.org -Port 80
Test-NetConnection vpn.myserver.duckdns.org -Port 443
```

兩個都應顯示 `TcpTestSucceeded : True`。這只能確認 TCP 可連，還需要實際用 HTTPS 網址確認憑證和管理頁。

若要確認 A1 本機的 Caddy 是否回應，在 A1 執行：

```bash
curl -I -H 'Host: vpn.myserver.duckdns.org' http://127.0.0.1
```

通常會收到 `308` 等轉址回應。本機有回應、外部卻連不上時，優先檢查 DNS、Oracle 規則及主機防火牆。不要直接清空 iptables。修正後 Caddy 會依日誌顯示的時間自動重試，不必反覆刪除憑證資料或重新部署。

**HTTPS 開得到，但顯示 `502 Bad Gateway`**

代表請求已到 Caddy，接著檢查 wg-easy 狀態、兩個容器是否都加入 `caddy` 網路，以及後端連接埠：

```bash
cd /opt/wg-easy
sudo docker compose ps
sudo docker compose logs --tail=50 wg-easy

cd /opt/caddy
sudo docker compose exec caddy sh -c 'wget -S -O /dev/null http://wg-easy:51821'
```

Caddy 和 wg-easy 必須共用 Docker 網路；**Caddy 容器裡的 `127.0.0.1` 是 Caddy 自己**，所以後端應填 `wg-easy:51821`。

**出現憑證警告**

確認使用的是 Caddyfile 設定的完整 HTTPS 網域，且 Caddyfile 沒有 `tls internal`。不要用 A1 公網 IP 代替網域測試這張網域憑證。

**其他啟動訊息**

`Caddyfile input is not formatted` 是排版提醒；`:80` 出現 HTTP/2／HTTP/3 因需要 TLS 而跳過，也不代表 HTTPS `443` 啟動失敗。以憑證申請結果和實際 HTTPS 存取判斷。

查看即時紀錄：

```bash
cd /opt/caddy
sudo docker compose logs --tail=50 -f caddy
```

按 `Ctrl+C` 只會退出日誌查看，不會停止服務。

## 8. wg-easy 初始化

第一次透過自己的 HTTPS 網址開啟管理頁後，依畫面建立管理帳號：

| 畫面項目 | 填寫內容 |
|---|---|
| Username | 自訂管理帳號 |
| Password／Confirm Password | 自訂管理密碼 |
| Existing Setup | 全新部署選 **No** |
| Host | **A1 的公網 IPv4** |
| Port | **51820** |

**Host 是 VPN 連線的目的地址**，不是瀏覽器開管理頁的地址。因此請填 A1 公網 IP，不要填 `127.0.0.1`、Docker 的 `10.42.42.42` 或 A1 的 Oracle 私網 IP。

`Existing Setup` 的匯入功能是針對專案支援的舊版本設定；不要把其他 WireGuard Docker 專案的設定檔當成可直接匯入的 wg-easy 備份。

完成後建議在右上角帳號選單的 **Account** 頁面啟用 2FA，並妥善保存恢復所需資訊。管理頁已可從外網連入，請使用獨立且足夠強的管理密碼。

來源：[wg-easy Setup](https://wg-easy.github.io/wg-easy/latest/guides/setup/)、[wg-easy 2FA](https://wg-easy.github.io/wg-easy/latest/guides/2fa/)。

## 9. 建立裝置與驗證 VPN

### 9.1 新增第一個手機裝置

在管理頁新增裝置，例如 `phone-test`。依畫面檢查或調整：

| 項目 | 初次測試建議 |
|---|---|
| Name | `phone-test` 或自己可辨識的名稱 |
| IPv4 address | 保留自動分配 |
| Allowed IPs | `0.0.0.0/0` |
| DNS | `1.1.1.1` |
| 其他參數 | 先保留預設 |

`Allowed IPs = 0.0.0.0/0` 代表所有 IPv4 目的位址都走此 VPN。它是用戶端路由設定，不是安全存取權限的完整保證。

### 9.2 手機匯入

1. 手機安裝官方 **WireGuard App**。
2. 在 App 新增通道，掃描電腦管理頁顯示的 QR Code。
3. 如果只有同一支手機可用，改用匯入該裝置的 `.conf` 檔案。
4. 關閉手機 Wi-Fi，使用行動網路測試。
5. 啟用通道，開啟一般網站確認能上網。

每台實體裝置都在 wg-easy 新增自己的 peer，分別使用自己的 `.conf`。不要讓手機與電腦同時共用同一組裝置金鑰，避免端點更新互相干擾。

### 9.3 確認 IPv4 出口

手機連線後，開啟：

**<https://api.ipify.org>**

顯示的 IPv4 應與 A1 的公網 IPv4 相同。可在 A1 執行以下指令比對：

```bash
curl -4 -fsS https://api.ipify.org
```

同時確認管理頁的裝置有近期交握與收送流量。也可以在 A1 查看：

```bash
sudo docker exec wg-easy wg show
```

看到 `latest handshake` 持續更新、收送量增加，再配合網站能正常使用及出口 IP 比對，才算完成本次 IPv4 驗證。App 只顯示開關已開，不代表隧道一定已連通。

### 9.4 Windows／WireSock

手機驗證成功後，另建一個裝置，例如 `windows`，下載它的 `.conf` 匯入 WireSock 或 WireGuard 客戶端。

如果目的只是讓遊戲走 VPN，可在 **WireSock 用戶端**使用 `AllowedApps`／Tunneled applications 白名單，只加入需要 VPN 的程式，讓其他程式維持原本網路。另確認 DNS 與 Kill Switch／Network Lock 的行為；程式分流不代表所有系統網路設定都跟著分開。

這些是家中 Windows 的 WireSock 設定，不會由 wg-easy 自動推送。新版本 WireSock 也可能在關閉視窗後保持 VPN 啟用，斷線時應使用連線開關。

來源：[WireGuard 官方下載](https://www.wireguard.com/install/)、[wg-easy Edit Client](https://wg-easy.github.io/wg-easy/latest/guides/clients/)、[WireSock 程式分流](https://wiresock.net/documentation/guide/route-one-app-through-wireguard-windows.html)、[WireSock Network Settings](https://wiresock.net/documentation/wiresock-secure-connect/network-settings.html)。

### 完成

至此就完成所有基本的設定，可以使用 VPN 服務了。

## 10. 日常管理與更新

wg-easy 與 Caddy 是兩個 Compose 專案，分別在 `/opt/wg-easy` 與 `/opt/caddy` 管理。

### 查看狀態與紀錄

```bash
cd /opt/wg-easy
sudo docker compose ps
sudo docker compose logs --tail=100 wg-easy

cd /opt/caddy
sudo docker compose ps
sudo docker compose logs --tail=100 caddy
```

### 修改設定

```bash
sudo nano /opt/wg-easy/compose.yaml
```

在 nano 中按 `Ctrl+O`、Enter 儲存，`Ctrl+X` 離開。

修改 Compose 後套用：

```bash
cd /opt/wg-easy
sudo docker compose config --quiet
sudo docker compose up -d
```

若變更導致容器重建，VPN 可能短暫中斷。單純 `restart` 不會套用所有 Compose 參數變更。

### 更新同一個 v15 標籤

先閱讀更新說明並備份持久資料，再於可接受短暫中斷的時間執行：

```bash
cd /opt/wg-easy
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
```

若改用固定版本標籤，先修改 `image` 的版本，再執行同樣命令。跨主要版本時依專案的 migration 文件處理。

### 修改或更新 Caddy

編輯設定：

```bash
sudo nano /opt/caddy/Caddyfile
```

只修改 Caddyfile 時，先驗證再重新載入：

```bash
cd /opt/caddy
sudo docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
```

若修改 Compose，使用 `sudo docker compose up -d` 套用。只更換 Caddyfile 的網域或轉發規則，則驗證後 reload 即可；新網域還需要正確 DNS 及憑證。更新 Caddy 映像則執行：

```bash
cd /opt/caddy
sudo docker compose pull
sudo docker compose up -d
```

### 停止與啟動

```bash
cd /opt/wg-easy
sudo docker compose stop
sudo docker compose start
```

**一般操作不要加 `down -v`**。`docker compose down -v` 會刪除這個 Compose 所管理的 named volume，可能讓管理帳號、裝置設定與金鑰消失。

### 確認 volume 並規劃備份

```bash
sudo docker inspect wg-easy --format '{{range .Mounts}}{{println .Type .Source "->" .Destination}}{{end}}'
sudo docker volume ls
```

找到指向 `/etc/wireguard` 的 volume，備份時應包含其完整內容，並考慮資料庫的一致性；只複製 `compose.yaml` 不夠。Caddy 另需保留 `compose.yaml`、`Caddyfile` 和實際掛載至 `/data` 的 volume；本教學預設通常為 `caddy_caddy_data`。備份含私鑰與登入資料，應存放於私人位置。

若資料夾或專案名稱變更，也要確認實際 volume 保持一致，避免連到新的空 volume。

來源：[Compose up](https://docs.docker.com/reference/cli/docker/compose/up/)、[Compose restart](https://docs.docker.com/reference/cli/docker/compose/restart/)、[Compose down](https://docs.docker.com/reference/cli/docker/compose/down/)、[Compose project name](https://docs.docker.com/compose/how-tos/project-name/)。

## 11. 日後調整

### 增加其他服務的子網域

同一個 DuckDNS 名稱可以繼續使用下一層名稱，例如 `files.myserver.duckdns.org`。讓新服務的 Compose 也以 external 方式加入 `caddy` 網路後，在 Caddyfile 新增該完整網域的 site block，轉發到那個服務的容器名稱與內部埠。新服務不需要加入 wg-easy 原本的 `wg` 網路。

DNS 解析、Caddy 路由與後端服務都需要設定正確。對一般 HTTP 網站使用反向代理，不代表 WireGuard 的 UDP VPN 流量也會自動經過 Caddy。

### 只允許特定來源或 VPN 內網管理

本文的管理入口對外提供 HTTPS。若日後改成只允許自己的來源 IP 或 VPN 裝置，需另外設計 Caddy 的存取規則、路由及憑證驗證方式；不要只關掉必要連接埠，導致後續憑證無法續期。

### 外網登入保護與代理資訊

wg-easy 官方文件說明它沒有內建登入請求速率限制。除強密碼與 2FA 外，可依需求參考官方的 Caddy 範例，加入適當的速率限制模組或 CrowdSec 等防護。

如果要讓 wg-easy 的安全紀錄顯示原始用戶端 IP，可依實際版本及部署設定 `TRUSTED_PROXIES`，只信任真正的 Caddy 來源地址；不要設成信任所有來源。本教學沒有先填入可能隨容器重建改變的代理 IP，也不把啟用此功能當成 HTTPS 運作的必要條件。

來源：[wg-easy Caddy 範例與速率限制說明](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/caddy/)、[wg-easy Optional Configuration](https://wg-easy.github.io/wg-easy/latest/advanced/config/optional-config/)。

## 12. 參考文件

- [wg-easy 專案](https://github.com/wg-easy/wg-easy)
- [wg-easy v15.5.0 Compose 原始範例](https://github.com/wg-easy/wg-easy/blob/v15.5.0/docker-compose.yml)
- [wg-easy v15.5.0 release](https://github.com/wg-easy/wg-easy/releases/tag/v15.5.0)
- [wg-easy Basic Installation](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/basic-installation/)
- [wg-easy Optional Configuration](https://wg-easy.github.io/wg-easy/latest/advanced/config/optional-config/)
- [wg-easy Caddy 範例](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/caddy/)
- [Caddy Automatic HTTPS](https://caddyserver.com/docs/automatic-https)
- [Caddy 官方容器映像](https://hub.docker.com/_/caddy)
- [DuckDNS](https://www.duckdns.org/)
- [DuckDNS API](https://www.duckdns.org/spec.jsp)
- [Compose 網路與跨專案連線](https://docs.docker.com/compose/how-tos/networking/)
- [Compose 網路介面與預設閘道](https://docs.docker.com/reference/compose-file/services/#networks)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Compose 專案名稱](https://docs.docker.com/compose/how-tos/project-name/)
- [Docker Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Oracle Security Lists](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securitylists.htm)
- [OCI 價目表](https://www.oracle.com/cloud/price-list/)
- [WireGuard 官方下載](https://www.wireguard.com/install/)
- [WireSock 文件](https://wiresock.net/documentation/wiresock-secure-connect/get-started.html)

