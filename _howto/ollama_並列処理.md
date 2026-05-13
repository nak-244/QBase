そのまま運用メモとして残せる形で整理します。
現在の構成・目的に合わせると、以下の形が実務上かなり分かりやすいです。

---

# Ollama 並列数・モデル保持設定変更手順（本番/開発共通）

## ■ 目的

Ollama の同時実行数とモデル保持数を調整し、
OpenWebUI + RAG 環境の安定性を改善する。

### 変更内容

```ini id="fh1r45"
OLLAMA_NUM_PARALLEL=2
OLLAMA_MAX_LOADED_MODELS=2
OLLAMA_KEEP_ALIVE=30m
```

---

# ■ 現在の構成

OpenWebUI（本番/開発）は Docker 上で動作しているが、
Ollama 本体はホスト側で共通利用している。

```text id="7m8uh7"
本番 OpenWebUI
  ↓
共通 Ollama

開発 OpenWebUI
  ↓
共通 Ollama
```

そのため、Ollama 設定変更は
本番・開発の両方へ反映される。

---

# ■ 各設定の意味

| 設定                         | 内容             |
| -------------------------- | -------------- |
| OLLAMA_NUM_PARALLEL=2      | 同時推論数を2に制限     |
| OLLAMA_MAX_LOADED_MODELS=2 | メモリ保持モデル数を2に制限 |
| OLLAMA_KEEP_ALIVE=30m      | 未使用モデルを30分保持   |

---

# ■ この設定にする理由

現在の環境：

* RTX 5060 Ti 8GB
* RAM 32GB
* OpenWebUI + RAG
* embedding 同時利用
* qwen3:14b / gemma3系利用

この構成では、

```text id="6a1xfr"
並列数を増やす
↓
VRAM圧迫
↓
RAMスワップ
↓
全体遅延
```

が発生しやすい。

そのため：

```text id="0kqb5q"
無理な並列化を抑え、
embedding と LLM を保持しながら
安定運用する
```

方向へ調整する。

---

# ■ 現在の設定確認

実行：

```bash id="juhw7j"
systemctl show ollama | grep OLLAMA
```

現在：

```text id="l8g4mb"
Environment="OLLAMA_HOST=0.0.0.0"
OLLAMA_NUM_PARALLEL=4
```

が設定されている。

---

# ■ 修正対象

編集対象：

```bash id="jlwm7g"
/etc/systemd/system/ollama.service.d/override.conf
```

本体 service は変更しない。

---

# ■ 設定変更手順

## ① override.conf 編集

実行：

```bash id="8z8f1k"
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```

---

## ② 現在の内容

```ini id="fb0ud6"
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_NUM_PARALLEL=4"
```

---

## ③ 下記へ変更

```ini id="6c3s0l"
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_NUM_PARALLEL=2"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
Environment="OLLAMA_KEEP_ALIVE=30m"
```

---

# ■ 保存方法（nano）

```text id="vx41h9"
Ctrl + O
Enter
Ctrl + X
```

---

# ■ systemd 再読込

実行：

```bash id="jlwm8m"
sudo systemctl daemon-reload
```

---

# ■ Ollama 再起動

実行：

```bash id="jlwm8x"
sudo systemctl restart ollama
```

---

# ■ 設定反映確認

実行：

```bash id="jlwm97"
systemctl show ollama | grep OLLAMA
```

正常時：

```text id="6bl01q"
OLLAMA_HOST=0.0.0.0
OLLAMA_NUM_PARALLEL=2
OLLAMA_MAX_LOADED_MODELS=2
OLLAMA_KEEP_ALIVE=30m
```

---

# ■ モデル保持確認

実行：

```bash id="jlwm9h"
ollama ps
```

例：

```text id="jlwm9r"
NAME
qwen3:14b
mxbai-embed-large
```

表示されれば、モデル保持が機能している。

---

# ■ 想定される効果

## 改善点

* VRAM暴走抑制
* RAMスワップ減少
* RAG応答安定化
* embedding再ロード減少
* 初回応答高速化

---

# ■ 注意点

gemma3:27b のような重量モデルを多重利用する場合は、
さらに：

```ini id="jlwmad"
OLLAMA_NUM_PARALLEL=1
```

まで下げる可能性がある。

---

# ■ 結論

現在のハードウェア構成では、

```ini id="jlwmam"
OLLAMA_NUM_PARALLEL=2
OLLAMA_MAX_LOADED_MODELS=2
OLLAMA_KEEP_ALIVE=30m
```

は、OpenWebUI + RAG + embedding 運用に対して、
安定性と実効速度のバランスが良い設定である。
