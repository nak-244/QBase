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

## ■ 完成状態

* [https://www.qbiworld.com](https://www.qbiworld.com) → OK
* [https://qbase.qbiworld.com](https://qbase.qbiworld.com) → OK（条件付き）

---
## ■ 自動起動フロー
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
cloudflare自動起動
↓
qbase.qbiworld.com公開
---

## ■ 注意点（重要）

* cloudflaredは**systemdで常駐済み**
* 手動軌道は不要
* wslが起動していない場合はqbase停止
* PC停止でqbase停止
+ Cloudflare機能（WAF等）は未使用構成
---
