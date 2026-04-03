# ■ 全体構成（理解）

まず構造を整理します。

* localhost:4000 → Docker（QBase）
* Cloudflare Tunnel → ローカルへ接続
* DNS → subdomainだけCloudflareへ

つまり👇

```
qbase.cosmegreed.com
   ↓（DNS）
Cloudflare
   ↓（Tunnel）
localhost:4000
```

---

# ■ STEP① Cloudflareアカウント＆ドメイン追加

### ① Cloudflareにログイン

[https://dash.cloudflare.com/](https://dash.cloudflare.com/)

### ② ドメイン追加

```
cosmegreed.com
```

### ③ プラン

→ FreeでOK

---

# ■ STEP② ネームサーバー変更（重要）

シンレンタルサーバーで取得しているため👇

### 変更先（Cloudflare指定のNS）

例：

```
xxxx.ns.cloudflare.com
yyyy.ns.cloudflare.com
```

### 作業場所

シンレンタルサーバー管理画面
→ ドメイン設定
→ ネームサーバー変更

---

# ■ この時点の注意

ここが最重要です ⚠️

👉 DNSをCloudflareに移すと

* Web
* メール

全部Cloudflare管理になる

---

# ■ STEP③ DNSを復元（壊さないため）

CloudflareのDNSに以下を設定

### ■ 既存サイト（維持）

```
Aレコード
@ → （今のサーバーIP）
```

### ■ メール（必須）

シンレンタルサーバーのMXをそのまま登録

例：

```
MX
mail.cosmegreed.com
```

※ここミスるとメール止まる

---

# ■ STEP④ Cloudflare Tunnelインストール（サーバー側）

WSL or Linuxで実行

### ① cloudflared インストール

```
sudo apt update
sudo apt install cloudflared
```

---

### ② ログイン

```
cloudflared tunnel login
```

→ ブラウザ開く
→ cosmegreed.com選択

---

# ■ STEP⑤ トンネル作成

指定通り👇

```
cloudflared tunnel create qbase-tunnel-cosmeg
```

作成される：

```
/home/xxx/.cloudflared/xxxxx.json
```

---

# ■ STEP⑥ DNSに紐付け

```
cloudflared tunnel route dns qbase-tunnel-cosmeg qbase.cosmegreed.com
```

👉 これでCNAME自動作成される

---

# ■ STEP⑦ 設定ファイル作成

```
nano ~/.cloudflared/config.yml
```

内容👇

```yaml
tunnel: qbase-tunnel-cosmeg
credentials-file: /home/xxx/.cloudflared/xxxxx.json

ingress:
  - hostname: qbase.cosmegreed.com
    service: http://localhost:4000
    originRequest:
      httpHostHeader: localhost

  - service: http_status:404
```

---

# ■ STEP⑧ 起動

```
cloudflared tunnel run qbase-tunnel-cosmeg
```

---

# ■ STEP⑨ 動作確認

```
https://qbase.cosmegreed.com
```

表示されれば成功

---

# ■ STEP⑩ 常駐化（必須）

```
sudo cloudflared service install
sudo systemctl start cloudflared
```

---

# ■ よくある失敗（重要）

### ❌ ① localhostにバインドしている

→ WSLの場合

```
OLLAMA_HOST=0.0.0.0
```

### ❌ ② Dockerが外から見えない

→ これはOK（Tunnelだから問題なし）

---

# ■ STEP⑪ 本番への展開（qbiworld）

そのまま横展開します。

### トンネル

```
qbase-tunnel
```

### コマンド

```
cloudflared tunnel create qbase-tunnel
cloudflared tunnel route dns qbase-tunnel qbase.qbiworld.com
```

### config.yml

```yaml
- hostname: qbase.qbiworld.com
  service: http://localhost:4000
```

---

# ■ 設計としての評価

この構成はかなり良いです。

理由👇

* ポート公開不要（セキュア）
* サーバーIP非公開
* Cloudflare WAF使える
* 証明書自動

---

# ■ 注意（本質的なリスク）

一点だけ正直に言います。

👉 DNSをCloudflareに寄せる以上
**メール設定を間違えると即死**

---

# ■ 補足（運用設計）

今のあなたの環境なら👇

* dev → localhost:4001
* prod → localhost:4000

なので将来的に👇

```
dev.qbase.xxx.com → 4001
qbase.xxx.com     → 4000
```

これも1トンネルで可能

---

# ■ 結論

* Cloudflare Tunnelで完全に実現可能
* DNS移行は必須（subdomainだけでは不可）
* メール設定の復元が最重要ポイント
* 構成としては本番運用レベルで問題なし

---
