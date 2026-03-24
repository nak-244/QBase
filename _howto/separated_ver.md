## ①公開環境：イメージ作成
公開用ソースへ移動：
```
cd Documents/GitHub/QBworld/14_QBase/_build/sauce/open-webui-source-v0.8.9-prod
```

ビルド：
```
docker buildx build --load -t qbase-prod:v0.8.9 .
```

#### ポイント
- ```prod``` ＝ 本番専用
- ```v0.8.9``` ＝ どのソースから作ったか追跡可能

## ② 公開環境 起動
```
docker rm -f open-webui
```

```
docker run -d \
  --restart unless-stopped \
  -p 3000:8080 \
  -e WEBUI_NAME="QBase" \
  -e OLLAMA_BASE_URL=http://172.28.109.111:11434 \
  -e WEBUI_DISABLE_VERSION_CHECK=true \
  -e ENABLE_OLLAMA_EMBEDDINGS=true \
  -e EMBEDDING_ENGINE=ollama \
  -e EMBEDDING_MODEL=mxbai-embed-large \
  -e USE_EMBEDDING_MODEL_DOCKER="" \
  -e RAG_EMBEDDING_MODEL="" \
  -e TZ=Asia/Tokyo \
  -v open-webui:/app/backend/data \
  --name open-webui \
  qbase-prod:v0.8.9
```

## ③ 開発環境：イメージ作成
```
cd Documents/GitHub/QBworld/14_QBase/_build/sauce/open-webui-source-v0.8.9-dev
```

ビルド：
```
docker buildx build --load -t qbase-dev:v0.8.9 .
```

## ④ 開発環境 起動
```
docker rm -f open-webui-dev
```

```
docker run -d \
  --restart unless-stopped \
  -p 4000:8080 \
  -e WEBUI_NAME="QBase-Dev" \
  -e OLLAMA_BASE_URL=http://172.28.109.111:11434 \
  -e WEBUI_DISABLE_VERSION_CHECK=true \
  -e ENABLE_OLLAMA_EMBEDDINGS=true \
  -e EMBEDDING_ENGINE=ollama \
  -e EMBEDDING_MODEL=mxbai-embed-large \
  -e USE_EMBEDDING_MODEL_DOCKER="" \
  -e RAG_EMBEDDING_MODEL="" \
  -e TZ=Asia/Tokyo \
  -v open-webui-dev:/app/backend/data \
  --name open-webui-dev \
  qbase-dev:v0.8.9
```

### 🔒 これで何が保証されるか
開発で何をしても：
| 操作         | 公開環境   |
| ---------- | ------ |
| 開発側ビルド     | ✅ 影響ゼロ |
| 開発側再起動     | ✅ 影響ゼロ |
| サーバー再起動    | ✅ 影響ゼロ |
| Docker自動復旧 | ✅ 影響ゼロ |

### 🎯 運用メリット
##### ■ ロールバック可能
例：
```
qbase-prod:v0.8.8
qbase-prod:v0.8.9
```
問題時に即戻せる

##### ■ 事故防止
開発中の壊れたビルドが公開に混入しない

##### ■ 監査性
「いつ・どの版を公開したか」証跡が残る

##### ■ タグ更新ルール
この手順は **初回 v0.8.9 リリース用** です。
修正時は下記のようにタグを進めます。

```
qbase-prod:v0.8.9
↓ 修正
qbase-prod:v0.8.9.1
↓ 追加修正
qbase-prod:v0.8.9.2
```
手順は同じで、タグだけ変更。

##### ■ 運用上のワンポイント
将来的に履歴が増えたら：
```
docker images
docker image prune
```
で整理すれば、容量管理も問題ありません。