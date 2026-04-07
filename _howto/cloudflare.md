# ■ Cloudflare Tunnel構成（標準モデル）

## ① 目的

ローカルアプリケーションを外部公開する

```text
https://qbase.qbiworld.com → localhost:4000
```

---

## ② アーキテクチャ

```text
ユーザー（ブラウザ）
↓ HTTPS
Cloudflare（SSL終端）
↓ Tunnel（cloudflared）
ローカルサーバー（localhost:4000）
↓
アプリケーション（OpenWebUI / Docker）
```

---

## ③ 構成要素

### ■ ドメイン管理

* Cloudflareで管理
* ネームサーバーをCloudflareへ委譲

---

### ■ DNS

* サブドメイン：`qbase.qbiworld.com`
* レコード：CNAME
* ターゲット：`<tunnel-id>.cfargotunnel.com`
* プロキシ：有効

---

### ■ Tunnel

#### 作成

```bash
cloudflared tunnel create qbase-tunnel
```

---

#### 設定（config.yml）

```yaml
tunnel: qbase-tunnel
credentials-file: /home/qpc/.cloudflared/xxxx.json

ingress:
  - hostname: qbase.qbiworld.com
    service: http://localhost:4000
    originRequest:
      httpHostHeader: localhost
  - service: http_status:404
```

---

#### 起動

```bash
cloudflared tunnel run qbase-tunnel
```

または常駐化

```bash
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

---

### ■ SSL（Cloudflare）

* モード：Full
* 証明書：Universal SSL（自動発行）

---

### ■ アプリケーション側

Dockerで公開

```bash
-p 4000:8080
```

👉 外部からは `localhost:4000` でアクセス

---

## ④ 通信の流れ（整理）

```text
https://qbase.qbiworld.com
↓
Cloudflare（証明書）
↓
cfargotunnel
↓
cloudflared
↓
http://localhost:4000
↓
アプリケーション
```

---

## ⑤ 特徴（設計思想）

* 公開ポート不要（443のみ）
* サーバーにグローバルIP不要
* Firewall開放不要
* Cloudflareでセキュリティ制御可能

---

## ⑥ この構成でできること

* ローカル開発環境の公開
* 社内ツールの外部アクセス
* APIの公開
* ゼロトラスト構成の実現

---

# 結論

👉 **Cloudflare Tunnelを用いた標準的な公開構成として成立している**

---
