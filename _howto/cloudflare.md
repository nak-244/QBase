## ■ Cloudflare Tunnel構築メモ（完成版）

### ■ 目的

ローカル環境（localhost:4000）を
外部URL（qbase.qbiworld.com）で公開する

---

## ■ 構成

```text
qbiworld.com           → 既存サーバー（GMO）
www.qbiworld.com       → 既存サーバー

qbase.qbiworld.com     → Cloudflare Tunnel
                       → localhost:4000（OpenWebUI）
```

---

## ■ DNS設定（GMO側）

* ネームサーバー：変更なし（GMOのまま）
* 追加設定：

```text
qbase → CNAME → <TunnelのUUID>.cfargotunnel.com
```

---

## ■ Cloudflare設定

* Tunnel作成済
* route dns 実行済

```bash
cloudflared tunnel create qbase-tunnel
cloudflared tunnel route dns qbase-tunnel qbase.qbiworld.com
```

---

## ■ config.yml

```yaml
tunnel: qbase-tunnel
credentials-file: /home/qpc/.cloudflared/<UUID>.json

ingress:
  - hostname: qbase.qbiworld.com
    service: http://localhost:4000
  - service: http_status:404
```

---

## ■ 起動方法

```bash
# 手動起動（不要）
# cloudflared tunnel run qbase-tunnel

# systemdで自動起動（現在）
sudo systemctl start cloudflared
```

---

## ■ 動作仕様（重要）

| 項目                      | 状態         |
| ----------------------- | ---------- |
| cloudflared（systemd）稼働中 | qbase表示される |
| cloudflared停止           | qbase停止    |
| PC起動                    | 自動でqbase復旧 |
| メインドメイン                 | 影響なし       |


---

## ■ メール設定

* すべてDNSのみ（グレー）
* Cloudflareプロキシは使用しない

---
##  ■ systemd設定
```bash
/etc/systemd/system/cloudflared.service

[Unit]
Description=Cloudflare Tunnel
After=network.target

[Service]
ExecStart=/usr/local/bin/cloudflared tunnel run qbase-tunnel
Restart=always
RestartSec=5
User=qpc

[Install]
WantedBy=multi-user.target
```
---
## ■ WSL自動起動
### ■ 設定内容
| 項目        | 設定                                         |
| --------- | ------------------------------------------ |
| トリガー      | ログオン時                                      |
| プログラム     | wsl                                        |
| 引数        | -d Ubuntu --exec bash -lc "sleep infinity" |
| 最上位の特権で実行 | 有効（推奨）                                     |

### ■ コマンド内容
```
wsl -d Ubuntu --exec bash -lc "sleep infinity"
```
* WSLを常時起動
* Systemdを維持
* cloudfraredの自動起動を成立

---

## ■ 完成状態

* [https://www.qbiworld.com](https://www.qbiworld.com) → OK
* [https://qbase.qbiworld.com](https://qbase.qbiworld.com) → OK（条件付き）

---
## ■ 自動起動フロー
```
PC起動
↓
Windowsログイン
↓
タスクスケジューラ
↓
WSL起動（常駐）
↓
systemd起動
↓
cloudflared自動起動（systemd）
↓
qbase.qbiworld.com公開
```
---
## ■ 動作仕様
| 状態     | 結果      |
| ------ | ------- |
| PC起動のみ | qbase停止 |
| ログイン後  | 自動起動    |
| WSL停止  | qbase停止 |


---
## ■ 注意点（重要）

* cloudflaredは**systemdで常駐済み**
* 手動起動は不要
* WSLが起動していない場合はqbase停止
* PC停止でqbase停止
+ Cloudflare機能（WAF等）は未使用構成
---
