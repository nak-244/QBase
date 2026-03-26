
# 本番用
本番用ソースへ移動：
```
cd /mnt/d/QBase/_build/source/open-webui-source-v0.8.10-prod
```

ビルド：
```
docker buildx build --load -t qbase-prod:v0.8.10 .
```
エラー発生時は--no-cacheをつける。
```
docker buildx build --no-cache --load -t qbase-prod:v0.8.10 .
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
  qbase-prod:v0.8.10
```

# 開発用
開発用ソースへ移動：
```
cd /mnt/d/QBase/_build/source/open-webui-source-v0.8.10-dev
```

ビルド：
```
docker buildx build --load -t qbase-dev:v0.8.10 .
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
  qbase-dev:v0.8.10
```
