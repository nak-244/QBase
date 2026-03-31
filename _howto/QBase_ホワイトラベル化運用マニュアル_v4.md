# QBase（OpenWebUIベース）ホワイトラベル化運用マニュアル【強化版】

------------------------------------------------------------------------

## 1. 目的

本手順は、OpenWebUIをベースにした社内AI基盤「QBase」を
自社ブランド仕様でビルド・管理・再構築できる状態を維持することを目的とする。

------------------------------------------------------------------------

## 2. 全体構成

| 区分 | 内容 |
| ---- | ---- |
| ベース | OpenWebUI v0.8.5 |
| カスタム名 | QBase |
| 実行環境 | Docker |
| LLM接続 | Ollama |
| 管理方法 | カスタムビルド |

------------------------------------------------------------------------

## 3. 推奨ディレクトリ構成

    QBase/
     ├─ favicon
     ├─ old
     ├─ open-webui-source-v85/
     └─ docker-run-v85.txt

------------------------------------------------------------------------

# 4. バージョンアップ(ソース修正)

## 4-1. ソース取得
[GitHub](https://github.com/open-webui/open-webui "GitHub") よりZIPをダウンロードし解凍。

## 4-2. ソースをQBaseディレクトリに保存
ディレクトリ名は open-webui-source-v085 のように、記述する。

## 4-3. サイト名変更
backend/open_webui/env.py
```
# WEBUI_NAME = os.environ.get("WEBUI_NAME", "Open WebUI")
#     if WEBUI_NAME != "Open WebUI":
#      WEBUI_NAME += " (Open WebUI)"
WEBUI_NAME = os.environ.get("WEBUI_NAME", "QBase")
```
上記のようにコメントアウトし、新たに記述を追加。

## 4-4. HTMLのlang属性・タイトル変更
src/app.html
```
<html lang="ja">
```
2行目のlang属性を `ja`に変更。

```
<title>QBase</title>
```
タイトル変更。

## 4-5. ロゴ変更
faviconフォルダ内のアイコンを、下記ディレクトリ内で差し替え
※ 修正箇所が追加される場合あり
- static/
- static/static/
- backend/open_webui/static/

## 4-6. Dockerfile修正
ルート/Dockerfile

### 38行目　これ不要かも
```COPY package.json package-lock.json ./``` の次に
```ENV NODE_OPTIONS="--max-old-space-size=4096"```
これを追加。

```
COPY package.json package-lock.json ./
ENV NODE_OPTIONS="--max-old-space-size=4096"
RUN npm ci --force
```

### 136行目あたり install python dependenciesブロック
```RUN pip install --no-cache-dir -r requirements.txt``` 追加。

```
# install python dependencies
COPY --chown=$UID:$GID ./backend/requirements.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
```

### 140行目あたり install python dependenciesブロック
下記に差し替え。
```
# RUN pip3 install --no-cache-dir uv && \
RUN if [ "$USE_CUDA" = "true" ]; then \
    pip3 install 'torch<=2.9.1' torchvision torchaudio --index-url https://download.pytorch.org/whl/$USE_CUDA_DOCKER_VER --no-cache-dir && \
    python -c "import os; from sentence_transformers import SentenceTransformer; SentenceTransformer(os.environ['RAG_EMBEDDING_MODEL'], device='cpu')" && \
    python -c "import os; from sentence_transformers import SentenceTransformer; SentenceTransformer(os.environ.get('AUXILIARY_EMBEDDING_MODEL', 'TaylorAI/bge-micro-v2'), device='cpu')" && \
    python -c "import os; from faster_whisper import WhisperModel; WhisperModel(os.environ['WHISPER_MODEL'], device='cpu', compute_type='int8', download_root=os.environ['WHISPER_MODEL_DIR'])" && \
    python -c "import os; import tiktoken; tiktoken.get_encoding(os.environ['TIKTOKEN_ENCODING_NAME'])" && \
    python -c "import nltk; nltk.download('punkt_tab')" \
    ; \
    else \
    pip3 install 'torch<=2.9.1' torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu --no-cache-dir && \
    if [ "$USE_SLIM" != "true" ]; then \
        python -c "import os; from sentence_transformers import SentenceTransformer; SentenceTransformer(os.environ['RAG_EMBEDDING_MODEL'], device='cpu')" && \
        python -c "import os; from sentence_transformers import SentenceTransformer; SentenceTransformer(os.environ.get('AUXILIARY_EMBEDDING_MODEL', 'TaylorAI/bge-micro-v2'), device='cpu')" && \
        python -c "import os; from faster_whisper import WhisperModel; WhisperModel(os.environ['WHISPER_MODEL'], device='cpu', compute_type='int8', download_root=os.environ['WHISPER_MODEL_DIR'])" && \
        python -c "import os; import tiktoken; tiktoken.get_encoding(os.environ['TIKTOKEN_ENCODING_NAME'])" && \
        python -c "import nltk; nltk.download('punkt_tab')" \
    ; \
    fi; \
    fi && \
    mkdir -p /app/backend/data && chown -R $UID:$GID /app/backend/data/ && \
    rm -rf /var/lib/apt/lists/*
```

## 4-7. 設定→概要部分非表示
src/lib/components/chat/SettingsModal.svelte
下記2か所削除

826行目
```
						{:else if tabId === 'about'}
							<button
								role="tab"
								aria-controls="tab-about"
								aria-selected={selectedTab === 'about'}
								class={`px-0.5 md:px-2.5 py-1 min-w-fit rounded-xl flex-1 md:flex-none flex text-left transition
								${
									selectedTab === 'about'
										? ($settings?.highContrastMode ?? false)
											? 'dark:bg-gray-800 bg-gray-200'
											: ''
										: ($settings?.highContrastMode ?? false)
											? 'hover:bg-gray-200 dark:hover:bg-gray-800'
											: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'
								}`}
								on:click={() => {
									selectedTab = 'about';
								}}
							>
								<div class=" self-center mr-2">
									<InfoCircle strokeWidth="2" />
								</div>
								<div class=" self-center">{$i18n.t('About')}</div>
							</button>
```

929行目
```
				{:else if selectedTab === 'about'}
					<About />
```

## 4-8. requirements.txt
backend/requirements.txt

```
pydub
ddgs==9.11.4
```
9.11.2 → 9.11.4に修正。


## 4-9. スレッドタイトル生成プロンプト変更
backend/open_webui/config.py
1710行目あたり
DEFAULT_TITLE_GENERATION_PROMPT_TEMPLATE = """### Task:
ここをブロックごと差し替え
```
DEFAULT_TITLE_GENERATION_PROMPT_TEMPLATE = """### Task:
Generate a concise short title with an emoji summarizing the chat history.

### Guidelines:
- The title should clearly represent the main theme or subject of the conversation.
- Use emojis that enhance understanding of the topic, but avoid quotation marks or special formatting.
- The title MUST be written in Japanese.
- Do NOT use English under any circumstances.
- Prioritize accuracy over excessive creativity; keep it clear and simple.
- Your entire response must consist solely of the JSON object, without any introductory or concluding text.
- The output must be a single, raw JSON object, without any markdown code fences or other encapsulating text.
- Ensure no conversational text, affirmations, or explanations precede or follow the raw JSON output, as this will cause direct parsing failure.

### Output:
JSON format: { "title": "your concise title here" }

### Examples:
- { "title": "🐛 タイトル生成不具合" },
- { "title": "🍪 チョコチップレシピ" },
- { "title": "音楽配信の進化" },
- { "title": "在宅勤務の生産性向上" },
- { "title": "医療と人工知能" },
- { "title": "🎮 ゲーム開発の知見" }

### Chat History:
<chat_history>
{{MESSAGES:END:2}}
</chat_history>"""
```



------------------------------------------------------------------------
