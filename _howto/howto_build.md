# 本番用
本番用ソースへ移動：
```
cd /mnt/d/QBase/_build/source/open-webui-source-v0.8.12-prod
```

ビルド：
```
docker buildx build --load -t qbase-prod:v0.8.12 .
```
エラー発生時は--no-cacheをつける。
```
docker buildx build --no-cache --load -t qbase-prod:v0.8.12 .
```

公開環境起動：
```
docker rm -f open-webui
```

```
docker run -d \
  --restart unless-stopped \
  -p 4000:8080 \
  -e WEBUI_NAME="QBase" \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  --add-host=host.docker.internal:host-gateway \
  -e WEBUI_DISABLE_VERSION_CHECK=true \
  -e ENABLE_OLLAMA_EMBEDDINGS=true \
  -e EMBEDDING_ENGINE=ollama \
  -e EMBEDDING_MODEL=mxbai-embed-large \
  -e USE_EMBEDDING_MODEL_DOCKER="" \
  -e RAG_EMBEDDING_MODEL="" \
  -e TZ=Asia/Tokyo \
  -v open-webui:/app/backend/data \
  --name open-webui \
  qbase-prod:v0.8.12
```

# 開発用
開発用ソースへ移動：
```
cd /mnt/d/QBase/_build/source/open-webui-source-v0.8.12-dev
```

ビルド：
```
docker buildx build --load -t qbase-dev:v0.8.12.1 .
```
エラー発生時は--no-cacheをつける。
```
docker buildx build --no-cache --load -t qbase-dev:v0.8.12.1 .
```

開発環境起動：
```
docker rm -f open-webui-dev
```

```
docker run -d \
  --restart unless-stopped \
  -p 4001:8080 \
  -e WEBUI_NAME="QBase" \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  --add-host=host.docker.internal:host-gateway \
  -e WEBUI_DISABLE_VERSION_CHECK=true \
  -e ENABLE_OLLAMA_EMBEDDINGS=true \
  -e EMBEDDING_ENGINE=ollama \
  -e EMBEDDING_MODEL=mxbai-embed-large \
  -e USE_EMBEDDING_MODEL_DOCKER="" \
  -e RAG_EMBEDDING_MODEL="" \
  -e TZ=Asia/Tokyo \
  -v open-webui-dev:/app/backend/data \
  --name open-webui-dev \
  qbase-dev:v0.8.12.1
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
