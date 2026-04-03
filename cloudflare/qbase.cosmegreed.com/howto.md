## 0. 先に全体像

最終的な着地点はこうです。

* `cosmegreed.com` → 既存のシンレンタルサーバーのサイトへ向ける
* `www.cosmegreed.com` → 同上
* メール関連（MX / SPF / DKIM / DMARC）→ 今のまま維持
* `qbase.cosmegreed.com` → Cloudflare Tunnel → `http://localhost:4000`

つまり、**CloudflareはDNSの権威サーバーになるが、Web本体は引き続きシンレンタルサーバー、QBaseだけTunnel** です。Cloudflare でドメインをフルセットアップする場合は、Cloudflare 側に必要なDNSレコードを先に揃えてから、レジストラ側でCloudflare指定のネームサーバーへ変更します。これは Cloudflare の公式なオンボーディング手順です。([Cloudflare Docs][2])

---

## 1. いま削除してはいけないもの

ここは先に明確にします。

**まだ削除しないもの**

* シンレンタルサーバー上の既存サイト設定
* 既存メール設定
* 既存DNS情報の参照元
* `qbase.cosmegreed.com` のシンレンタルサーバー側サブドメイン設定

最後のサブドメイン設定は、**切替完了後は不要になる可能性が高い**です。
ただし、今はまだ削除しません。先に消す理由がありませんし、切替前に消しても得はありません。

---

## 2. Cloudflare に `cosmegreed.com` を追加

Cloudflare ダッシュボードで次を行います。

1. Cloudflare にログイン
2. 「Add a domain」または「Add site」
3. `cosmegreed.com` を入力
4. プランは通常 Free で足ります
5. 進むと、Cloudflare が既存DNSを自動スキャンします

この自動スキャン結果は**信用し切らないでください**。
抜けが普通にあります。ここから先は、**シンレンタルサーバー側の現在のDNS設定を正として、Cloudflareに手で照合**するのが安全です。Cloudflare 側では、DNSレコードはゾーンに追加・編集でき、最終的にCloudflareの割り当てたネームサーバーをレジストラ側へ設定して有効化します。([Cloudflare Docs][3])

---

## 3. シンレンタルサーバー側のDNSを全部洗い出す

ここが最重要です。
**この工程が雑だと事故になります。**

シンレンタルサーバーのDNS設定画面で、現在のレコードを全部確認してください。対象は最低でも次です。

* A
* AAAA
* CNAME
* MX
* TXT
* CAA
* SRV（使っていれば）
* DKIM用CNAMEまたはTXT
* SPF
* DMARC
* 自動設定された mail / www / ftp など

この時点でやることは、**Cloudflare に同じ内容を作る**ことです。
Cloudflare へ移す対象は「Webだけ」ではありません。**メール周りも含めたDNS全体**です。Cloudflare にネームサーバーを切り替えるということは、以後 Cloudflare がそのドメインの権威DNSになるからです。([Cloudflare Docs][2])

---

## 4. Cloudflare 側で既存レコードを再現する

Cloudflare の DNS Records 画面で、シンレンタルサーバー側のレコードを1件ずつ転記します。

ここでのルールは重要です。

### 4-1. メインサイトのA/CNAME

`cosmegreed.com` や `www.cosmegreed.com` が今どこを向いているかを、そのまま Cloudflare に入れます。

### 4-2. メール系レコード

MX / SPF / DKIM / DMARC は**そのまま**入れます。
ここを1つでも落とすと、メール受信不能、送信失敗、迷惑メール判定の原因になります。

### 4-3. Proxy の考え方

Cloudflare には各DNSレコードに「プロキシあり / DNS only」があります。

原則はこうです。

* `qbase.cosmegreed.com` の Tunnel 用 CNAME → **プロキシあり**
* メール系（MX、mail用A/CNAME、DKIMなど）→ **DNS only**
* メインサイト系は状況次第
  まず安全優先なら、最初は **DNS only** で切り替え、その後必要なら Cloudflare Proxy を有効化する方法が堅いです

理由は、まず「切替後も従来どおり動く」を優先するためです。
最初から余計な最適化を入れると、切り分けが難しくなります。

---

## 5. `qbase.cosmegreed.com` 用の Tunnel 側設定を作る

Cloudflare Tunnel では、公開ホスト名をローカルサービスへ割り当てます。Cloudflare 公式でも、公開ホスト名を `localhost:8080` 等へ向ける形が基本です。([Cloudflare Docs][4])

Cloudflare ダッシュボード側でトンネルを作る場合の流れは次です。

1. Cloudflare Zero Trust に入る
2. `Networks` → `Connectors` → `Cloudflare Tunnels`
3. `Create a tunnel`
4. コネクタ種別は `Cloudflared`
5. トンネル名を付ける
   例: `qbase-tunnel`
➡qbase-tunnel-cosmegreed
Cloudflare 公式でも、トンネル作成はこの画面経由で案内されています。([Cloudflare Docs][5])

---

## 6. サーバー側で `cloudflared` をそのトンネルに接続

これは QBase を動かしているマシン側で行います。
もしすでに別ドメイン向けで `cloudflared` が使えていたなら、流れは同じです。

典型的には次の流れです。

```bash
cloudflared login
cloudflared tunnel create qbase-tunnel
```

その後、認証ファイルが作られます。
Cloudflare 公式でも、`cloudflared` を認証し、トンネルを作成し、設定ファイルを置いて実行する流れが基本です。([Cloudflare Docs][6])

---

## 7. `config.yml` を作る

QBase 側の転送先は `http://localhost:4000` です。
設定はこうします。

```yaml
tunnel: qbase-tunnel
credentials-file: /home/＜ユーザー名＞/.cloudflared/＜トンネルUUID＞.json

ingress:
  - hostname: qbase.cosmegreed.com
    service: http://localhost:4000
  - service: http_status:404
```

補足です。

* `hostname` は公開したい名前
* `service` はローカル側の実サービス
* 最後の `http_status:404` は必須に近いです
  該当しないHostヘッダのアクセスを落とすためです

---

## 8. DNS とトンネルを結び付ける

このコマンドをサーバー側で実行します。

```bash
cloudflared tunnel route dns qbase-tunnel qbase.cosmegreed.com
```

Cloudflare 公式では、このコマンドは**トンネルへ向くDNS CNAMEを作る**用途として案内されています。([Cloudflare Docs][7])

実行後、Cloudflare の DNS Records に

* Type: `CNAME`
* Name: `qbase`
* Target: `＜トンネルUUID＞.cfargotunnel.com`

が作成されれば正常です。ダッシュボードから手動作成してもよく、Cloudflare のDNSレコードとしてはこの形が公式です。([Cloudflare Docs][8])

---

## 9. まだネームサーバーは変えずに、ローカル側だけ先に確認

ここで一度、サーバー側の QBase 自体を確認します。

まず OpenWebUI コンテナが起動していることを確認します。
運用メモでは `open-webui` コンテナがホストの 4000 番で公開される構成です。

確認コマンド例です。

```bash
docker ps
curl http://localhost:4000
```

ここで HTML が返れば、QBase 側は正常です。
ここで失敗するなら、Cloudflare 以前の問題です。先にローカルを直します。

---

## 10. `cloudflared` を起動してトンネル単体を確認

次にトンネルを起動します。

```bash
cloudflared tunnel run qbase-tunnel
```

systemd 管理にするなら恒久運用できますが、最初はまず手動起動で十分です。
Cloudflare Tunnel は、`cloudflared` が外向き接続を張る方式なので、公開IPを直接開ける必要はありません。これは Cloudflare Tunnel の基本仕様です。([Cloudflare Docs][1])

---

## 11. ここでようやくネームサーバー変更

Cloudflare でゾーンが整い、必要レコードが揃い、トンネル設定も済んだら、最後にレジストラ側でネームサーバーを変更します。

今回は `cosmegreed.com` はシンレンタルサーバーで取得とのことなので、**シンレンタルサーバーのドメイン管理画面**で、Cloudflare が表示した 2つのネームサーバーへ変更します。

Cloudflare の公式でも、フルセットアップでは**レジストラ側で既存ネームサーバーをCloudflareの割当ネームサーバーへ置き換える**流れです。Cloudflare ブランドのネームサーバー名は自動割当で、任意にはできません。([Cloudflare Docs][2])

ここで初めて、権威DNSが Cloudflare に切り替わります。

---

## 12. 反映待ち

ネームサーバー変更は即時ではありません。
反映まで時間差があります。

確認対象は次です。

* `cosmegreed.com`
* `www.cosmegreed.com`
* メール送受信
* `qbase.cosmegreed.com`

Cloudflare 側でゾーンが Active になり、ブラウザや `nslookup` / `dig` でも Cloudflare 側の情報が見え始めれば切替完了です。([Cloudflare Docs][2])

---

## 13. 切替後の確認

ここは順番にやってください。

### 13-1. メインサイト

* `https://cosmegreed.com/`
* `https://www.cosmegreed.com/`

従来どおり表示されるか確認します。

### 13-2. qbase

* `https://qbase.cosmegreed.com/`

QBase が表示されれば成功です。

### 13-3. メール

実際に

* 外部から自分宛てに受信
* 自分から外部へ送信

の両方を確認します。
MX が正しくても、SPF/DKIM/DMARC が崩れていると送信側で問題が出ることがあります。

### 13-4. DNS確認

例です。

```bash
nslookup qbase.cosmegreed.com
nslookup -type=mx cosmegreed.com
nslookup -type=txt cosmegreed.com
```

`qbase.cosmegreed.com` は最終的に Tunnel へ向く CNAME が見えればよいです。

---

## 14. シンレンタルサーバー側の `qbase` サブドメインはその後どうするか

切替完了後、`qbase.cosmegreed.com` は Cloudflare DNS で管理され、実体は Cloudflare Tunnel によって提供されます。
したがって、**シンレンタルサーバー側での「qbase サブドメイン登録」は通常は不要**です。

ただし、削除タイミングは**全部の確認が終わった後**です。
今すぐ消す理由はありません。

---

## 15. 典型的な失敗点

ここは非常に重要です。

* Cloudflare に MX を入れ忘れる
  → メール受信不可
* SPF / DKIM / DMARC の TXT/CNAME を移し忘れる
  → 送信失敗、迷惑メール判定
* `cosmegreed.com` / `www` のA/CNAMEが不一致
  → メインサイトが落ちる
* `qbase` のDNSだけ作って `cloudflared` を起動していない
  → `qbase` だけ開かない
* `service` を `http://localhost:4000` ではなく間違ったポートにする
  → 502 / 523系エラーになりやすい
* いきなりネームサーバー変更する
  → 切戻しが面倒になる

---

## 16. 今回の最短で安全な実行順

実務上は、この順でやるのが安全です。

1. Cloudflare に `cosmegreed.com` を追加
2. シンレンタルサーバーのDNSレコードを全部洗い出す
3. Cloudflare に既存DNSを完全再現
4. `qbase` 用トンネルを作成
5. `config.yml` に `qbase.cosmegreed.com -> http://localhost:4000` を設定
6. `cloudflared tunnel route dns qbase-tunnel qbase.cosmegreed.com`
7. `curl http://localhost:4000` でローカル確認
8. `cloudflared tunnel run qbase-tunnel` でトンネル起動
9. 最後にシンレンタルサーバー側でネームサーバーをCloudflareへ変更
10. メインサイト・qbase・メールを順に確認
11. 問題なければシンレンタル側の `qbase` サブドメイン設定を整理

---

## 17. 今回の判断

この件で本質的に重要なのは、**Cloudflare Tunnel の設定そのものより、DNS移管時の既存レコード移植漏れを防ぐこと**です。
Tunnel 自体は難しくありません。難所は DNS 全体の整合です。

結論として、あなたの進め方は正しいです。
やるべきことは、**Cloudflare側に既存DNSを完全移植してから、最後にネームサーバーを切り替える**ことです。その上で `qbase.cosmegreed.com` だけを Tunnel に向ければ、`cosmegreed.com` 本体とメールを維持したまま QBase を公開できます。([Cloudflare Docs][2])

次に進めるために、必要であれば「シンレンタルサーバーで確認すべきDNSレコード一覧」と「Cloudflareに1件ずつどう転記するか」を、そのまま作業メモとして使える形で作成します。

[1]: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/?utm_source=chatgpt.com "Cloudflare Tunnel · Cloudflare One docs"
[2]: https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/?utm_source=chatgpt.com "Change your nameservers (Full setup) · Cloudflare DNS docs"
[3]: https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/?utm_source=chatgpt.com "Manage DNS records"
[4]: https://developers.cloudflare.com/learning-paths/clientless-access/connect-private-applications/create-tunnel/?utm_source=chatgpt.com "Create a Cloudflare Tunnel · Cloudflare Learning Paths"
[5]: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/?utm_source=chatgpt.com "Create a tunnel (dashboard) · Cloudflare One docs"
[6]: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/create-local-tunnel/?utm_source=chatgpt.com "Create a locally-managed tunnel"
[7]: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/tunnel-useful-commands/?utm_source=chatgpt.com "Useful commands · Cloudflare One docs"
[8]: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/routing-to-tunnel/dns/?utm_source=chatgpt.com "DNS records · Cloudflare One docs"
