
## ■ 全体像

```text
https://dev.example.co.jp
        ↓
Cloudflare Tunnel
        ↓
東京PC localhost:4000
```

---

## ■ 前提

* サブドメイン：例）`dev.example.co.jp`
* 東京PCで開発環境稼働：`http://localhost:4000`

---

## ■ 手順

### STEP1：Cloudflare にログイン

[https://dash.cloudflare.com/](https://dash.cloudflare.com/)

---

### STEP2：サブドメインを“サイト追加”

1. 「Add a site」
2. 入力：

   ```
   dev.example.co.jp
   ```
3. Freeプラン選択

---

### STEP3：ネームサーバー確認

表示される：

```
xxxx.ns.cloudflare.com
yyyy.ns.cloudflare.com
```

👉 これは「サブドメイン専用ネームサーバー」

---

### STEP4：iCLUSTA+ 側で委任設定（重要）

DNS設定で **NSレコード追加**

| 種別 | ホスト名 | 値                      |
| -- | ---- | ---------------------- |
| NS | dev  | xxxx.ns.cloudflare.com |
| NS | dev  | yyyy.ns.cloudflare.com |

これで：

```
dev.example.co.jp のDNS管理 → Cloudflareへ移譲
```

※ 本体ドメインは無関係

---

### STEP5：Cloudflare側 DNS設定

Cloudflare管理画面
→ DNS → Records → 追加

| Type  | Name | Target                 |
| ----- | ---- | ---------------------- |
| CNAME | dev  | xxxxx.cfargotunnel.com |

（トンネル作成後に設定）

---

### STEP6：東京PCに cloudflared 導入

公式DL：
[https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/)

Windows版インストール

---

### STEP7：Cloudflareログイン（東京PC）

PowerShell：

```powershell
cloudflared tunnel login
```

ブラウザが開く → 認証

---

### STEP8：トンネル作成

```powershell
cloudflared tunnel create qbase-dev
```

---

### STEP9：設定ファイル作成

作成：

```
C:\Users\ユーザー名\.cloudflared\config.yml
```

内容：

```yaml
tunnel: qbase-dev
credentials-file: C:\Users\ユーザー名\.cloudflared\xxxxx.json

ingress:
  - hostname: dev.example.co.jp
    service: http://localhost:4000
  - service: http_status:404
```

---

### STEP10：トンネル起動

```powershell
cloudflared tunnel run qbase-dev
```

---

## ■ 完了

大阪からアクセス：

```
https://dev.example.co.jp
```

✔ HTTPS
✔ 固定URL
✔ 警告ページなし
✔ 高速
✔ 無料

---

## ■ 任意：アクセス制限（推奨）

Cloudflare Zero Trust → Access

* Googleログイン必須
* 社内メール限定
* IP制限

---

## ■ よくある確認

| 項目      | 状態 |
| ------- | -- |
| 本体サイト影響 | なし |
| メール影響   | なし |
| SEO影響   | なし |
| URL固定   | 可  |
| 常設運用    | 可  |

---
