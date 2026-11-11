# 自建 VPN 完整教學：用 wg-easy 架設 WireGuard 伺服器

從只有 SSH 可以登入的 Ubuntu A1 主機開始，完成 Docker 安裝、Oracle 網路設定、wg-easy 初始化與手機 VPN 連線測試。管理介面先透過 SSH 通道存取，日後可再調整為 VPN 內網或 HTTPS 網址。

本文以 **2026-11-11** 查閱的 **wg-easy v15.5.0 專案範例**為基準，部署路徑統一使用 **`/opt/wg-easy`**。本文所稱「官方範例」是指 **wg-easy 專案自己發布的範例**；wg-easy 是第三方專案，並非 WireGuard 官方推出的管理介面。

> 適用範圍：全新的 Oracle Cloud A1、Ubuntu 24.04 LTS、ARM64、已有公網 IPv4，且可以用 `ubuntu` 帳號與 SSH 私鑰登入。本文先驗證 IPv4 VPN，不涵蓋完整的 IPv6 對外連線設定。
>

## 目錄

1. [部署方式與準備事項](#1-部署方式與準備事項)
2. [確認主機環境](#2-確認主機環境)
3. [安裝 Docker Engine 與 Compose](#3-安裝-docker-engine-與-compose)
4. [Oracle 網路規則](#4-oracle-網路規則)
5. [建立 Compose 設定](#5-建立-compose-設定)
6. [啟動與檢查服務](#6-啟動與檢查服務)
7. [透過 Windows SSH 通道開啟管理頁](#7-透過-windows-ssh-通道開啟管理頁)
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
| 管理介面 | A1 本機 `127.0.0.1:51821`，由 SSH 通道存取 |
| Docker 內部網路 | 沿用範例 `10.42.42.0/24` 與 IPv6 ULA 網段 |
| wg-easy 容器 IPv4 | 沿用範例 `10.42.42.42` |
| VPN 測試範圍 | IPv4；本文設定 `DISABLE_IPV6=true` |
| 持久資料 | 沿用範例的 `etc_wireguard` named volume |

這裡有三種不同的位址：

- **A1 公網 IP**：手機／電腦建立 VPN 連線時使用。
- **Docker 容器 IP**，例如 `10.42.42.42`：Docker 內部使用，不是手機連線時填寫的伺服器地址。
- **VPN 裝置 IP**：裝置連上 VPN 後使用，由 wg-easy 設定分配。

### 需要先準備

- A1 的公網 IPv4。
- 能登入 Oracle Console、修改主機所屬網路規則的權限。
- SSH 私鑰，以及能登入 A1 的 SSH 客戶端。
- 一台可安裝 WireGuard App 的手機，用行動網路測試。

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

### 4.2 新增 VPN 的 ingress rule

在 Oracle Console 找到 A1 使用的子網路及 Security List。常見路徑：

**Networking → Virtual cloud networks → 你的 VCN → Subnets → 使用中的公用子網路 → Security lists → 使用中的 Security List → Add ingress rules**。

新增：

| 欄位 | 設定 |
|---|---|
| Stateless | 不勾選，使用 stateful 規則 |
| Source Type | CIDR |
| Source CIDR | `0.0.0.0/0` |
| IP Protocol | UDP |
| Source Port Range | All／留空 |
| Destination Port Range | `51820` |
| Description | `WireGuard VPN` |

保留原有 SSH 與其他必要規則。本文假設沿用允許一般對外流量的 egress 設定；若有另外限制 egress 或主機防火牆，也需要確認 VPN 所需流量能通過。

Security List 規則會套用至與它關聯的子網路；也可以依自己的管理方式，改用掛在該 A1 VNIC 上的 NSG。

**此部署不需要對外開放 TCP `51821`**。這是管理頁的連接埠，稍後會限制在 A1 本機。

來源：[Oracle Security Lists](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securitylists.htm)、[wg-easy Basic Installation](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/basic-installation/)。

## 5. 建立 Compose 設定

### 5.1 建立目錄

```bash
sudo mkdir -p /opt/wg-easy
cd /opt/wg-easy
```

### 5.2 寫入設定

以下保留 wg-easy v15.5.0 原始範例的網路、固定容器 IP 與主要容器選項。整段貼上，包含最後的 `EOF`；結束標記 `EOF` 必須單獨一行。

```bash
sudo tee /opt/wg-easy/compose.yaml >/dev/null <<'EOF'
volumes:
  etc_wireguard:

services:
  wg-easy:
    environment:
      - INSECURE=true
      - DISABLE_IPV6=true

    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    networks:
      wg:
        ipv4_address: 10.42.42.42
        ipv6_address: fdcc:ad94:bacf:61a3::2a
    volumes:
      - etc_wireguard:/etc/wireguard
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
      - "127.0.0.1:51821:51821/tcp"
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
EOF
```

### 5.3 和原始範例的差異

| 項目 | 原始範例 | 本文 | 原因 |
|---|---|---|---|
| HTTP 管理頁 | 預設 `INSECURE=false` | `INSECURE=true` | 允許在 SSH 加密通道內使用 HTTP |
| 管理頁映射 | `51821:51821/tcp` | `127.0.0.1:51821:51821/tcp` | 管理入口只綁定 A1 本機 |
| VPN IPv6 | 預設啟用 | `DISABLE_IPV6=true` | 本教學先驗證 IPv4 VPN |

其他核心配置保留原樣。Docker 內部 IPv6 網路仍沿用範例，但 `DISABLE_IPV6` 讓 wg-easy 不替 VPN 介面與裝置啟用 IPv6；兩者是不同層的網路設定。依專案文件，即使管理頁仍顯示 IPv6 欄位，也不代表它已在 VPN 中使用。

**這不會關閉手機或電腦原有的 IPv6。** 本文只能確認 IPv4 是否經 A1 轉送；若要求 IPv6 也經 VPN，需要另行完成並驗證整條 IPv6 路徑。

來源：[v15.5.0 原始 Compose](https://github.com/wg-easy/wg-easy/blob/v15.5.0/docker-compose.yml)、[Optional Configuration](https://wg-easy.github.io/wg-easy/latest/advanced/config/optional-config/)、[Docker port publishing](https://docs.docker.com/engine/network/port-publishing/)、[Docker volumes](https://docs.docker.com/engine/storage/volumes/)。

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

測試 A1 本機管理頁：

```bash
curl -I http://127.0.0.1:51821
```

收到正常 HTTP 回應，例如成功或重新導向，代表網頁服務有回應。若容器持續重新啟動或有錯誤，先看紀錄，不要先改 Oracle 管理頁連接埠規則。

> Docker 的 bridge 網路會建立轉送、連接埠映射與 NAT 規則。本文沿用 Docker 預設網路管理方式，沒有清空 Oracle 映像的防火牆，也沒有停用 Docker 的防火牆管理。不要把「清空所有 iptables」當成通用解法。

來源：[Docker packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/)。

## 7. 透過 Windows SSH 通道開啟管理頁

### 7.1 建立通道

在**自己的 Windows 電腦**開啟 PowerShell，使用能登入 A1 的私鑰：

```powershell
# 換成你自己的私鑰完整路徑與 A1 公網 IPv4。
$keyPath = "C:\Users\你的帳號\.ssh\ssh-key.key"
$serverIp = "203.0.113.10"

ssh -i "$keyPath" -o ExitOnForwardFailure=yes -N -L 127.0.0.1:51821:127.0.0.1:51821 "ubuntu@$serverIp"
```

`203.0.113.10` 是文件用的示範 IP，務必替換。

第一次連線可能要求確認主機指紋。可透過已信任的 A1 SSH 連線或主控台，取得主機的 ED25519 公鑰指紋：

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

確認和 Windows 顯示的 `SHA256:...` 相同後，再輸入 `yes`。

通道成功後，PowerShell 會保持開著，通常沒有額外輸出；`-N` 表示不開啟互動式 shell，這是正常現象。

在自己電腦的瀏覽器開啟：

**<http://127.0.0.1:51821>**

SSH 通道只供目前這台電腦使用；手機的 `127.0.0.1` 指的是手機自己，不能用相同網址直接存取電腦上開的通道。

### 7.2 如果出現 Windows 私鑰權限錯誤

常見錯誤：

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions for '...ssh-key.key' are too open.
This private key will be ignored.
Permission denied (publickey).
```

這表示 Windows OpenSSH 不願使用權限過寬的私鑰。以下指令只處理指定的私鑰檔案；在 Windows PowerShell 執行，第一行換成實際路徑：

```powershell
$keyPath = "C:\Users\你的帳號\.ssh\ssh-key.key"
$accountName = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name

icacls "$keyPath" /setowner "$accountName"
icacls "$keyPath" /grant:r "${accountName}:R"
icacls "$keyPath" /inheritance:r
icacls "$keyPath" /remove:g "*S-1-5-32-545" "*S-1-5-11" "*S-1-1-0"

icacls "$keyPath"
```

這會保留目前帳號的讀取權限、移除繼承權限，並移除 `Users`、`Authenticated Users`、`Everyone` 等群組的明確授權。使用 SID 可避免中英文 Windows 群組名稱不同的問題。

如果顯示拒絕存取，用**同一個 Windows 帳號的系統管理員 PowerShell**再處理。若修正後仍顯示 `Bad permissions`，檢查最後的 ACL 輸出是否還有其他使用者的授權。若權限錯誤消失但仍是 `Permission denied (publickey)`，再核對私鑰、A1 IP 與登入帳號是否正確。

修正權限通常只做一次；日後仍需在每次管理時開啟 SSH 通道。關閉通道不會停止 wg-easy 或 VPN。

來源：[Win32 OpenSSH 檔案權限說明](https://github.com/PowerShell/Win32-OpenSSH/wiki/Security-protection-of-various-files-in-win32-openssh)、[Microsoft icacls](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls)。

## 8. wg-easy 初始化

第一次開啟管理頁後，依畫面建立管理帳號：

| 畫面項目 | 填寫內容 |
|---|---|
| Username | 自訂管理帳號 |
| Password／Confirm Password | 自訂管理密碼 |
| Existing Setup | 全新部署選 **No** |
| Host | **A1 的公網 IPv4** |
| Port | **51820** |

**Host 是 VPN 連線的目的地址**，不是瀏覽器開管理頁的地址。因此請填 A1 公網 IP，不要填 `127.0.0.1`、Docker 的 `10.42.42.42` 或 A1 的 Oracle 私網 IP。

`Existing Setup` 的匯入功能是針對專案支援的舊版本設定；不要把其他 WireGuard Docker 專案的設定檔當成可直接匯入的 wg-easy 備份。

完成後可啟用管理帳號的 2FA，並妥善保存恢復所需資訊。

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

### 查看狀態與紀錄

```bash
cd /opt/wg-easy
sudo docker compose ps
sudo docker compose logs --tail=100 wg-easy
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

找到指向 `/etc/wireguard` 的 volume，備份時應包含其完整內容，並考慮資料庫的一致性；只複製 `compose.yaml` 不夠。備份含私鑰與登入資料，應存放於私人位置。

若資料夾或專案名稱變更，也要確認實際 volume 保持一致，避免連到新的空 volume。

來源：[Compose up](https://docs.docker.com/reference/cli/docker/compose/up/)、[Compose restart](https://docs.docker.com/reference/cli/docker/compose/restart/)、[Compose down](https://docs.docker.com/reference/cli/docker/compose/down/)、[Compose project name](https://docs.docker.com/compose/how-tos/project-name/)。

## 11. 日後調整

### 改成 VPN 內網管理

之後可以調整管理頁的綁定地址、路由與防火牆，只讓 VPN 裝置存取。本文目前綁在 `127.0.0.1`，因此手機只是連上 VPN，並不會自動取得這個 localhost 管理入口。

### 改成外網 HTTPS 管理

可依專案文件加入 Caddy 或其他反向代理、網域及適當的雲端防火牆規則。屆時重新檢查 `INSECURE` 與代理信任設定，保留同一個 VPN volume 即可。

若使用 Caddy 的公開網域與自動公開憑證，依照 Caddy Automatic HTTPS 文件設定。專案 Caddy 範例若含 `tls internal`，使用的是 Caddy 內部 CA，不能直接當成所有瀏覽器自動信任的公開憑證。

來源：[wg-easy Caddy 範例](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/caddy/)、[Caddy Automatic HTTPS](https://caddyserver.com/docs/automatic-https)。

## 12. 參考文件

- [wg-easy 專案](https://github.com/wg-easy/wg-easy)
- [wg-easy v15.5.0 Compose 原始範例](https://github.com/wg-easy/wg-easy/blob/v15.5.0/docker-compose.yml)
- [wg-easy v15.5.0 release](https://github.com/wg-easy/wg-easy/releases/tag/v15.5.0)
- [wg-easy Basic Installation](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/basic-installation/)
- [wg-easy Optional Configuration](https://wg-easy.github.io/wg-easy/latest/advanced/config/optional-config/)
- [wg-easy No Reverse Proxy](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/reverse-proxyless/)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Compose 專案名稱](https://docs.docker.com/compose/how-tos/project-name/)
- [Docker Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Oracle Security Lists](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securitylists.htm)
- [OCI 價目表](https://www.oracle.com/cloud/price-list/)
- [WireGuard 官方下載](https://www.wireguard.com/install/)
- [WireSock 文件](https://wiresock.net/documentation/wiresock-secure-connect/get-started.html)
