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
RUN npm run buildをコメントアウト。下記のように変更する。
```
# RUN npm run build
RUN NODE_OPTIONS="--max-old-space-size=4096" npm run build
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

------------------------------------------------------------------------
