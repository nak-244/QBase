承知しました。
今回は「検証用の `qbase.cosmegreed.com` を Cloudflare Tunnel で `http://localhost:4000` に向ける」ことを、**最初から順番に、飛ばさず**進めます。`localhost:4000` で公開環境の QBase が動いている前提は、手元の起動メモとも一致しています。

まず結論だけ先に整理します。

`qbase.cosmegreed.com` のサブドメインを、今のシンレンタルサーバー側で事前作成する必要はありません。今回のやり方では、**Cloudflare 側で zone を管理し、Cloudflare 側 DNS に `qbase` を作る**流れです。Cloudflare Tunnel は `cloudflared` を入れて認証し、named tunnel を作成し、設定ファイルを置き、公開先 hostname を結び、最後にサービス化するのが公式の基本手順です。([Cloudflare Docs][1])

ただし、`cosmegreed.com` を Cloudflare に載せるということは、**ネームサーバーを Cloudflare に切り替える**ことを意味します。Free/Pro の一般的な構成では full setup になり、既存サイト用の A/CNAME、メール用の MX・TXT なども Cloudflare 側へ正しく再現する必要があります。ここを落とすと、既存サイトやメールに影響が出ます。([Cloudflare Docs][2])

では、順番に進めます。今回は「今やること」と「その画面で何を確認するか」を明確に書きます。

まず Phase 0 です。ここは実作業前の前提確認です。

今の状態として必要なのは次の4点です。
1つ目、`http://localhost:4000` にブラウザでアクセスすると QBase が表示されること。
2つ目、その 4000 番はサーバー上で実際に応答していること。
3つ目、Cloudflare アカウントを作成済み、または作成できること。
4つ目、シンレンタルサーバーのドメイン管理画面に入れることです。

ここで大事なのは、**`qbase.cosmegreed.com` をシンレンタルサーバーで先に作らない**ことです。不要です。むしろ今は余計なことを足さない方が安全です。

次に Phase 1、「今の DNS 情報を退避する」です。
これは必須です。理由は、Cloudflare にネームサーバーを切り替えると、今までシンレンタルサーバー側で持っていた DNS 情報を Cloudflare 側で引き継ぐ必要があるからです。Cloudflare もドメイン追加後に DNS レコードのレビューを求めています。([Cloudflare Docs][2])

やることは以下です。

シンレンタルサーバーの管理画面に入り、`cosmegreed.com` の現在の DNS レコードを確認してください。確認対象は最低でも次です。

A または AAAA レコード
CNAME レコード
MX レコード
TXT レコード
必要があれば SPF, DKIM, DMARC に相当する TXT

ここは「見えるものを全部控える」が正解です。
もし画面上に一覧表示があるなら、**画面全体のスクリーンショットを残す**のが安全です。
特に MX と TXT を取りこぼすと、メール障害の原因になります。

次が Phase 2、「Cloudflare に `cosmegreed.com` を追加する」です。

Cloudflare ダッシュボードにログインし、`Add a domain` から `cosmegreed.com` を追加します。Cloudflare の公式手順では、apex domain を入力して onboarding し、DNS レコードを確認し、その後 nameserver を変更する流れです。([Cloudflare Docs][2])

この段階で Cloudflare は、既存 DNS をある程度自動検出して候補表示することがあります。
ただし、**自動検出を信用し切ってはいけません**。Phase 1 で控えた内容と突き合わせてください。

Cloudflare 側でこの時点で作るべきものは、少なくとも次です。

既存サイト表示用の A または CNAME
メール用の MX
メール関連 TXT
そして今回追加する予定の `qbase` 用レコード

ただし、`qbase` だけは tunnel 作成後に自動生成させてもよいです。

Phase 3 は「ネームサーバー切替前の DNS 完成確認」です。

この時点では、まだシンレンタルサーバーのネームサーバーは変えません。
Cloudflare DNS 画面を開き、**既存サイトが表示されるためのレコード**と**メール用レコード**がそろっているか確認します。Cloudflare の DNS レコード作成・編集は Records 画面から行います。([Cloudflare Docs][3])

ここでの判断基準は単純です。
「今シンレンタルサーバー側で見えている DNS レコードが、Cloudflare 側にも同じ意味で並んでいるか」です。

次に Phase 4、「サーバー側に cloudflared を入れる」です。

Cloudflare の公式では、origin server に `cloudflared` をインストールし、認証し、named tunnel を作る手順です。([Cloudflare Docs][1])

Ubuntu / Debian 系なら、まず Cloudflare の Linux 用インストール手順に従います。公式は OS とアーキテクチャに応じた導入を案内しています。([Cloudflare Docs][4])

実作業の考え方はこうです。
サーバーに `cloudflared` コマンドが入る
↓
`cloudflared tunnel login` で Cloudflare アカウント認証
↓
`cloudflared tunnel create qbase-tunnel-cosmeg` で named tunnel 作成
↓
設定ファイル作成
↓
DNS と hostname を結ぶ
↓
サービス化

ここで、以前の別ドメイン運用と混ざらないように、今回の検証トンネル名は予定通り **`qbase-tunnel-cosmeg`** にします。これは正しい切り分けです。

Phase 5 は「Cloudflare 認証」です。

サーバー上で次を実行します。

```bash
cloudflared tunnel login
```

するとブラウザ認証画面が開き、対象 zone を選びます。ここでは `cosmegreed.com` を選択します。公式も、ローカル管理 tunnel の作成前に認証が必要としています。([Cloudflare Docs][1])

認証が成功すると、`~/.cloudflared/` 配下に証明用ファイルが作られます。

Phase 6 は「tunnel 作成」です。

```bash
cloudflared tunnel create qbase-tunnel-cosmeg
```

これで tunnel UUID と資格情報 JSON が発行されます。公式の named tunnel 作成手順に沿っています。([Cloudflare Docs][5])

Phase 7 は「設定ファイル作成」です。

`~/.cloudflared/config.yml` を作ります。内容は次で問題ありません。

```yaml
tunnel: qbase-tunnel-cosmeg
credentials-file: /home/ユーザー名/.cloudflared/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX.json

ingress:
  - hostname: qbase.cosmegreed.com
    service: http://localhost:4000
    originRequest:
      httpHostHeader: localhost
  - service: http_status:404
```

ここで `credentials-file` のパスだけは、実際に生成された JSON 名に置き換えてください。
`service: http://localhost:4000` は、今の QBase 公開コンテナ設定と一致しています。

`httpHostHeader: localhost` は、アプリ側が Host ヘッダを気にする場合の保険として入れてよい設定です。なくても動く場合はありますが、入れておく方が無難です。

Phase 8 は「DNS ルートを結ぶ」です。

Cloudflare 公式のローカル管理 tunnel では、トンネル作成後に公開 hostname をルーティングします。([Cloudflare Docs][5])
今回のコマンドはこれです。

```bash
cloudflared tunnel route dns qbase-tunnel-cosmeg qbase.cosmegreed.com
```

これで Cloudflare DNS 側に `qbase.cosmegreed.com` 用レコードが作られます。
つまり、ここで初めて `qbase` サブドメインが DNS 上に現れます。
だから、冒頭の「まだ qbase サブドメイン設定を何もしていない」は、むしろ正常です。

Phase 9 は「一時起動で動作確認」です。

```bash
cloudflared tunnel run qbase-tunnel-cosmeg
```

別ターミナルで `https://qbase.cosmegreed.com/` へアクセスします。
ただし、この時点では**ネームサーバー切替後でないと外部から正しく引けない**ので、順番は厳密に言うと次の Phase 10 と前後します。
つまり、DNS がまだシンレンタルサーバー側のままなら、Cloudflare 側に作った `qbase` はまだ世界から見えません。

なので実際の順番は、
Cloudflare 側 DNS 完成
↓
シンレンタルサーバーで nameserver を Cloudflare 指定値に変更
↓
伝播確認
↓
tunnel 実行確認
です。

Phase 10 は「シンレンタルサーバーでネームサーバー変更」です。

Cloudflare にドメイン追加後、Cloudflare が 2 つの assigned nameservers を表示します。Cloudflare の full setup 手順でも、その nameservers を registrar 側へ設定する流れです。([Cloudflare Docs][6])

シンレンタルサーバー側では、ドメインのネームサーバー設定画面から、その 2 つへ変更します。シンレンタルサーバー系のマニュアル群にも、ネームサーバー設定の導線があります。([Shin Server][7])

ここで重要なのは、**切替前に Cloudflare 側 DNS が完成していること**です。
未完成のまま切り替えると、既存サイトやメールが止まります。

Phase 11 は「反映確認」です。

ネームサーバー変更後、確認するものは3系統です。

1つ目、`https://cosmegreed.com/` が従来通り表示されること。
2つ目、メール送受信が従来通りであること。
3つ目、`https://qbase.cosmegreed.com/` が QBase を表示すること。

DNS 反映は即時とは限りません。
Cloudflare でも nameserver 変更後の検証を行う流れです。([Cloudflare Docs][6])

Phase 12 は「サービス化」です。

Cloudflare は `cloudflared` を OS サービスとして動かすことを推奨しています。Linux の場合、named tunnel と `config.yml` の準備後に service 化します。([Cloudflare Docs][8])

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

これで OS 起動時に自動復旧しやすくなります。

ここまでを踏まえて、今この瞬間にあなたがやるべき最初の作業は1つです。

**シンレンタルサーバー側の現在の DNS レコードを、A / CNAME / MX / TXT を漏れなく控えること**です。
これをやらずに先へ行くのは危険です。`qbase` サブドメインが未設定なのは問題ではありません。問題になるのは、既存サイトとメールのレコード取りこぼしです。

次の返信では、その Phase 1 を前提に、
**「シンレンタルサーバー画面で何を見ればいいか」
「Cloudflare に何をどう写せばよいか」
「どのレコードは触らず、どれを追加するか」**
の実務手順だけに絞って、さらに細かく進めます。

[1]: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/?utm_source=chatgpt.com "Set up your first tunnel · Cloudflare One docs"
[2]: https://developers.cloudflare.com/fundamentals/manage-domains/add-site/?utm_source=chatgpt.com "Onboard a domain · Cloudflare Fundamentals docs"
[3]: https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/?utm_source=chatgpt.com "Manage DNS records"
[4]: https://developers.cloudflare.com/tunnel/setup/?utm_source=chatgpt.com "Set up Cloudflare Tunnel"
[5]: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/create-local-tunnel/?utm_source=chatgpt.com "Create a locally-managed tunnel"
[6]: https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/?utm_source=chatgpt.com "Change your nameservers (Full setup) · Cloudflare DNS docs"
[7]: https://www.shin-server.jp/support/manual/man_all.php?utm_source=chatgpt.com "全マニュアル一覧 | 新世代レンタルサーバー ..."
[8]: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/as-a-service/linux/?utm_source=chatgpt.com "Run as a service on Linux"
