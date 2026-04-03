1. 事前確認です。今回の到達点は次の構成です。

* `https://cosmegreed.com/` は現状維持
* `https://qbase.cosmegreed.com/` だけを Cloudflare Tunnel 経由で `http://localhost:4000` に向ける
* トンネル名は `qbase-tunnel-cosmeg`

構造はこうなります。

```text
qbase.cosmegreed.com
  ↓ DNS（Cloudflare）
Cloudflare Tunnel
  ↓
http://localhost:4000
```

2. Cloudflare に `cosmegreed.com` を追加します。
   Cloudflare公式では、まずドメインを追加し、既存DNSを確認し、その後に割り当てられたネームサーバーへ変更する流れです。([Cloudflare Docs][3])

操作です。

* Cloudflare にログイン
* 「Add a domain」または「Onboard a domain」
* `cosmegreed.com` を入力
* プランは Free で開始

このあと Cloudflare が既存DNSレコードをスキャンします。

3. ここが最重要です。
   Cloudflare が拾ってきたDNSを**必ず確認・補完**してください。ネームサーバーをCloudflareに切り替えると、その後は Cloudflare 側のDNS設定が正になります。つまり、**今動いているWeb用のA/CNAME、メール用のMX/TXT/SPF/DKIM/DMARCを落とすと既存運用が壊れます**。([Cloudflare Docs][1])

この段階で確認すべき主なレコードは次です。

* `@` の A もしくは CNAME
* `www` の CNAME
* `mail` の A または CNAME
* MX
* TXT（SPF）
* DKIM 用TXTまたはCNAME
* `_dmarc` のTXT
* 必要ならFTP等のホスト名

判断基準です。

* `cosmegreed.com` 本体を今のまま表示したい
  → 既存の `@` と `www` を Cloudflare DNS にそのまま再現
* メールを今のまま使いたい
  → MX, SPF, DKIM, DMARC を必ず再現
* 今回新規に追加するのは `qbase` サブドメインだけ

4. 次に、Cloudflare Tunnel 用の `cloudflared` を接続先マシンへ入れます。
   Cloudflare 公式でも、ローカル管理のトンネルは `cloudflared` のインストール → 認証 → トンネル作成 → 設定ファイル作成 → ルーティング → 起動、の順です。([Cloudflare Docs][4])

Ubuntu / WSL なら、最も確実なのは Cloudflare ダッシュボードまたは公式配布に出るインストール手順に従うことです。([Cloudflare Docs][5])
画面からコピペするのが安全ですが、まずはインストール確認コマンドの形だけ示します。

```bash
cloudflared --version
```

これで未導入なら、Cloudflare公式の Downloads 画面の Ubuntu/Debian 手順で入れてください。([Cloudflare Docs][5])

5. `cloudflared` を Cloudflare アカウントへ認証します。
   ローカル管理トンネルでは `cloudflared tunnel login` を実行してブラウザ認証します。([Cloudflare Docs][4])

```bash
cloudflared tunnel login
```

これを実行するとブラウザが開きます。
そこで Cloudflare にログインし、対象ゾーンとして `cosmegreed.com` を選択します。認証が通ると、ローカルに認証用ファイルが作られます。([Cloudflare Docs][4])

6. トンネルを作成します。
   トンネル名はご指定どおり `qbase-tunnel-cosmeg` にします。トンネル作成時に資格情報JSONが生成されます。([Cloudflare Docs][4])

```bash
cloudflared tunnel create qbase-tunnel-cosmeg
```

成功すると、概ね次のような情報が出ます。

* Tunnel UUID
* credentials file の保存先
  例: `/home/ユーザー名/.cloudflared/<UUID>.json`

この **UUID と JSON のパスは後で使うので控えてください**。

7. トンネル設定ファイルを作成します。
   Cloudflare公式のローカル管理トンネルでも、`config.yml` を作って hostname と local service を結びます。([Cloudflare Docs][4])

作成先の例です。

```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

内容は次です。`credentials-file` だけは、実際に作成されたJSONファイル名へ置き換えてください。

```yaml
tunnel: qbase-tunnel-cosmeg
credentials-file: /home/ユーザー名/.cloudflared/ここにUUID.json

ingress:
  - hostname: qbase.cosmegreed.com
    service: http://localhost:4000
  - service: http_status:404
```

補足です。
今回は `localhost:4000` を公開先にしています。これは今のQBase本番コンテナのポート公開設定 `-p 4000:8080` と一致しています。

8. 次に、Cloudflare 側DNSへ `qbase.cosmegreed.com` をトンネルへ紐付けます。
   ローカル管理トンネルでは `cloudflared tunnel route dns` で公開ホスト名を作れます。([Cloudflare Docs][4])

```bash
cloudflared tunnel route dns qbase-tunnel-cosmeg qbase.cosmegreed.com
```

これにより、Cloudflare DNS に `qbase.cosmegreed.com` 用のレコードが追加されます。
この作業は、**ゾーン `cosmegreed.com` がすでに Cloudflare 管理になっていること**が前提です。

9. トンネルを起動して動作確認します。
   Cloudflare公式のローカル管理トンネルでも `cloudflared tunnel run` で起動します。([Cloudflare Docs][4])

```bash
cloudflared tunnel run qbase-tunnel-cosmeg
```

この状態で別ブラウザから、

```text
https://qbase.cosmegreed.com
```

へアクセスします。

ここで表示されれば、トンネル自体は成功です。

10. 常駐化します。
    検証中は手動起動でもよいですが、実運用では `cloudflared` をサービス化した方が安定します。Cloudflare は常駐運用を前提に案内しています。([Cloudflare Docs][6])

環境によって実装は少し違いますが、一般には次のどちらかです。

* Cloudflare が案内する service install 手順を使う
* systemd ユニットを自前で作る

まずは簡易に試すなら、起動確認後に service install 系の案内に沿って常駐化してください。Cloudflare のトンネル管理画面でも該当コマンドが提示されることがあります。([Cloudflare Docs][5])

11. ここで、ネームサーバー変更をまだしていない場合の作業です。
    Cloudflare でドメイン追加後、Cloudflare から **割り当てられた2つのネームサーバー** が表示されます。Cloudflareのフルセットアップでは、その2つへドメイン登録側を変更します。([Cloudflare Docs][1])

シンレンタルサーバー側では、公式マニュアルどおりドメインのネームサーバー設定画面から変更します。([shin-server.jp][2])

流れです。

* シンアカウントへログイン
* 対象ドメイン `cosmegreed.com` を選択
* ネームサーバー設定を開く
* Cloudflare から指定された2つのNSへ変更
* 保存

変更後、反映待ちになります。

12. 反映確認です。
    Cloudflare公式でも、ネームサーバー変更後は Cloudflare 側でアクティブになるまで待ち、確認する流れです。([Cloudflare Docs][1])

反映後に確認するものはこの3つです。

* `https://cosmegreed.com/` が従来どおり表示されるか
* 既存メールの送受信が正常か
* `https://qbase.cosmegreed.com/` が QBase を表示するか

ここで `cosmegreed.com` 本体かメールが壊れた場合は、Cloudflare DNS に既存レコードの不足・誤りがあります。Tunnel の問題ではなく、**ゾーン移行時のDNS再現ミス**です。

13. 本番 `qbase.qbiworld.com` への展開前提です。
    今回の検証が通れば、本番は同じ設計でできます。違うのは次の2点だけです。

* ドメインが `qbiworld.com`
* トンネル名が `qbase-tunnel`

つまり本番側は、概念的には次の差し替えです。

```bash
cloudflared tunnel create qbase-tunnel
cloudflared tunnel route dns qbase-tunnel qbase.qbiworld.com
```

`config.yml` はこうなります。

```yaml
tunnel: qbase-tunnel
credentials-file: /home/ユーザー名/.cloudflared/本番UUID.json

ingress:
  - hostname: qbase.qbiworld.com
    service: http://localhost:4000
  - service: http_status:404
```

14. 実務上の注意です。
    今回のやり方は適切ですが、前回つまずいた本質は「Cloudflare Tunnelが難しい」のではなく、**既存ドメインをCloudflare管理へ移す際のDNS再現を甘く見ると事故る**という点です。
    Tunnel 自体はシンプルです。難所は DNS とメールです。そこだけ丁寧にやれば問題ありません。⚠️

ここまでを、実際の作業単位でさらに細かく区切ると次の順です。

* まず Cloudflare に `cosmegreed.com` を追加
* Cloudflare が拾った DNS を全部確認
* 既存Webとメールのレコードを補完
* `cloudflared` を導入
* `cloudflared tunnel login`
* `cloudflared tunnel create qbase-tunnel-cosmeg`
* `config.yml` 作成
* `cloudflared tunnel route dns ...`
* ネームサーバーをシンレンタルサーバーで Cloudflare に変更
* 動作確認
* 常駐化
