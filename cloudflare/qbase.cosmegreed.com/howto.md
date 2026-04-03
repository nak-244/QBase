# ■ Cloudflare Tunnel構築 手順書（完全版）

---

# ■ 1. 目的

ローカル環境で稼働するQBase（OpenWebUI）を、外部からHTTPSでアクセス可能にする。

---

# ■ 2. システム構成

```text
ユーザー（ブラウザ）
↓
Cloudflare（DNS + HTTPS）
↓
Cloudflare Tunnel
↓
ローカルサーバー（localhost:4000）
```

---

# ■ 3. 前提条件

* DockerでQBaseが起動している

  ```bash
  http://localhost:4000
  ```



* ドメイン取得済み（例：cosmegreed.com）

---

# ■ 4. DNS設定（Cloudflare）

## ■ 4-1 ドメイン追加

Cloudflareにドメインを追加

---

## ■ 4-2 ネームサーバー変更

シンレンタル側でCloudflare指定NSに変更

---

## ■ 4-3 DNSレコード設定

| 種別  | 名前  | 値                | 備考  |
| --- | --- | ---------------- | --- |
| A   | @   | 162.43.96.26     | 本体  |
| A   | www | 162.43.96.26     |     |
| A   | *   | 162.43.96.26     |     |
| MX  | @   | sv6005.wpx.ne.jp | メール |
| TXT | @   | SPF設定            |     |

---

## ■ 4-4 注意点

* MXを誤るとメール停止
* NS変更後は反映待ち（最大24h）

---

# ■ 5. Cloudflare Tunnel構築

---

## ■ 5-1 cloudflaredインストール

```bash
sudo apt update
sudo apt install cloudflared
```

---

## ■ 5-2 認証

```bash
cloudflared tunnel login
```

※既にcert.pemがある場合はスキップ

---

## ■ 5-3 トンネル作成

```bash
cloudflared tunnel create qbase-tunnel-cosmeg
```

---

## ■ 生成ファイル

```text
/home/qpc/.cloudflared/
  ├ cert.pem
  ├ config.yml
  └ c4990ee2-xxxx.json
```

---

## ■ 5-4 DNS紐付け

```bash
cloudflared tunnel route dns qbase-tunnel-cosmeg qbase.cosmegreed.com
```

---

# ■ 6. config.yml（詳細解説）

```yaml
tunnel: qbase-tunnel-cosmeg
credentials-file: /etc/cloudflared/c4990ee2-5afd-4b5c-83c5-b455707e880a.json

ingress:
  - hostname: qbase.cosmegreed.com
    service: http://localhost:4000
    originRequest:
      httpHostHeader: localhost

  - service: http_status:404
```

---

## ■ 各項目の意味（重要）

### ■ tunnel

```yaml
tunnel: qbase-tunnel-cosmeg
```

→ Cloudflare側で作成したトンネル名

---

### ■ credentials-file

```yaml
credentials-file: /etc/cloudflared/xxxx.json
```

→ 認証ファイル（トンネル固有）

※サービス化時は必ず `/etc/cloudflared` に置く

---

### ■ ingress（ルーティング設定）

---

### ■ hostname

```yaml
hostname: qbase.cosmegreed.com
```

→ このドメインでアクセスされた場合

---

### ■ service

```yaml
service: http://localhost:4000
```

→ ローカルの転送先

---

### ■ originRequest

```yaml
httpHostHeader: localhost
```

→ Hostヘッダをlocalhostとして送る

※OpenWebUIなどで重要

---

### ■ fallback

```yaml
- service: http_status:404
```

→ 該当しないアクセスは404

---

# ■ 7. サービス化（本番運用）

---

## ■ 7-1 ディレクトリ作成

```bash
sudo mkdir -p /etc/cloudflared
```

---

## ■ 7-2 ファイルコピー

```bash
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/
sudo cp ~/.cloudflared/*.json /etc/cloudflared/
```

---

## ■ 7-3 config修正

```bash
sudo nano /etc/cloudflared/config.yml
```

---

### ■ 修正内容

```yaml
credentials-file: /etc/cloudflared/xxxx.json
```

---

## ■ 7-4 サービス登録

```bash
sudo cloudflared service install
```

---

## ■ 7-5 起動

```bash
sudo systemctl start cloudflared
```

---

## ■ 7-6 状態確認

```bash
systemctl status cloudflared
```

---

# ■ 8. 動作確認

---

## ■ 8-1 ローカル

```bash
curl http://localhost:4000
```

---

## ■ 8-2 DNS

```bash
nslookup -type=ns cosmegreed.com
```

---

## ■ 8-3 サブドメイン

```bash
nslookup qbase.cosmegreed.com
```

---

## ■ 8-4 外部アクセス

```bash
https://qbase.cosmegreed.com
```

---

# ■ 9. トラブル対応

---

## ■ 9-1 Tunnel起動してない

```bash
sudo systemctl restart cloudflared
```

---

## ■ 9-2 DNS未反映

→ 時間待ち

---

## ■ 9-3 localhost死んでる

→ Docker再起動

---

# ■ 10. 本番展開（次ステップ）

同手順で以下に展開可能

```text
qbase.qbiworld.com
```

---

# ■ 最終構成

```text
Cloudflare → Tunnel → localhost:4000（QBase）
```

---

# ■ 結論

👉 この構成で外部公開・常時稼働が成立
👉 config.ymlが中核（ルーティング定義）
👉 この手順はそのまま横展開可能

---
