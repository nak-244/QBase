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
cloudflared tunnel run qbase-tunnel
```

---

## ■ 動作仕様（重要）

| 項目             | 状態         |
| -------------- | ---------- |
| cloudflared起動中 | qbase表示される |
| cloudflared停止  | qbase停止    |
| メインドメイン        | 影響なし       |

---

## ■ メール設定

* すべてDNSのみ（グレー）
* Cloudflareプロキシは使用しない

---

## ■ 完成状態

* [https://www.qbiworld.com](https://www.qbiworld.com) → OK
* [https://qbase.qbiworld.com](https://qbase.qbiworld.com) → OK（条件付き）

---

## ■ 注意点（重要）

* cloudflaredは常駐必須
* PC停止でqbase停止
* Cloudflare機能（WAF等）は未使用構成

---

## ■ 今後の改善候補

* systemdで常駐化
* 別サーバーに移行
* Cloudflare完全移管

---

# ■ 結論

👉 **構成は完成済み（実運用可能）**
👉 **次フェーズは「運用最適化」**

---
