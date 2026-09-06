> **状態:** 過去の記録（2026-05-04時点）。現在の構成とは異なります。
> - モデル: Qwen3-14B → **Qwen3.8-27B (MLX)**（2026-08）
> - Aider: **2026-08-25 に破棄**（理由は aibou-plan/decisions/decision_log.md）
> - 価値があるのは「Ollama をなぜ選ばなかったか」という選択の理由の部分

# ローカルLLM環境構築記録：LM Studio + Qwen + Aider

## 最終構成

| 役割 | ツール |
|------|--------|
| モデルサーバー | LM Studio（OpenAI互換APIサーバー） |
| モデル | Qwen3-14B-MLX-4bit（コーディング全般） |
| コーディングAI | Aider |
| マシン | MacBook Air / ROG Flow Z13（Apple Silicon / iGPU） |

---

## 選択に至るまでの思考過程

### Phase 1：Ollamaから始める

最初の選択肢はOllama。理由はシンプルさ。

- CLIだけで完結
- `ollama run qwen3.5:9b` 一発で動く
- Aider・Claude Code両方から呼べる

**課題として見えてきたこと：**

- 思考モード（`/think`）が有効だと出力が遅く、Aider連携では邪魔になる
  - 対策として `qwen35_nothink.modelfile` を作成し `/no_think` をプロンプトに埋め込む
- モデルの管理UIがなく、複数モデルの比較・切り替えが不便
- Ollama 0.20.7以降の `ollama launch` は便利だが、Mac Apple Silicon での MLX 最適化は非対応

### Phase 2：LM Studio へ移行

LM Studioを選んだ理由：

1. **MLX対応**：Apple Silicon Mac では MLX バックエンドで推論が高速になる（OllamaのlllamaよりMLX Qwen3-14Bの方が体感速度が上）
2. **OpenAI互換APIサーバー**：Aiderの `--openai-api-base` でそのまま向けられる
3. **モデル管理GUIがある**：複数モデルを試しやすい
4. **`separateReasoningContentInAPI: true`** が設定されており、thinking contentが構造化されてAPIに返ってくる

### Phase 3：モデル選定

試したモデル（いずれもMLX 4bit量子化）：

| モデル | 印象 |
|--------|------|
| `DeepSeek-R1-Distill-Qwen-14B-4bit` | 思考が長い。コード生成より推論向き |
| `Qwen2.5-Coder-14B-Instruct-MLX-4bit` | コード特化、安定、早い |
| `Qwen3-14B-MLX-4bit` | バランスよく汎用。`/no_think` で思考抑制可能 |

**採用：Qwen3-14B-MLX-4bit**（Aiderとの相性◎、日本語も扱える）

---

## 環境構築手順（再構築用）

### 1. LM Studio インストール

[lmstudio.ai](https://lmstudio.ai) からダウンロード・インストール。

### 2. モデルのダウンロード

LM Studio の「Discover」タブで以下を検索・DL：

```
mlx-community/Qwen3-14B-MLX-4bit
```

バックアップ候補（コード特化）：
```
lmstudio-community/Qwen2.5-Coder-14B-Instruct-MLX-4bit
```

### 3. LM Studio ローカルAPIサーバーを起動

「Developer」タブ → 「Start Server」

- デフォルトポート：`http://localhost:1234`
- OpenAI互換エンドポイント：`http://localhost:1234/v1`

モデルをロード（UIでモデルを選択してロード）。

### 4. Aider インストール

```bash
pip install aider-chat
# または
pipx install aider-chat
```

### 5. Aider から LM Studio に接続

```bash
aider \
  --model openai/qwen3-14b \
  --openai-api-base http://localhost:1234/v1 \
  --openai-api-key lm-studio
```

`--openai-api-key` の値は何でもよい（LM Studioはキー不要）。

`.aider.conf.yml` に固定する場合：

```yaml
analytics: false
model: openai/qwen3-14b
openai-api-base: http://localhost:1234/v1
openai-api-key: lm-studio
```

### 6. 動作確認

```bash
aider --no-git
# > /ask 自己紹介して
```

---

## Qwen3 の思考モード制御

Qwen3は `<think>` トークンで思考を出力する。Aider連携では通常オフにした方が速い。

**LM Studio 側で無効化（推奨）：**
モデルのシステムプロンプトに追加するか、モデルの設定で「Reasoning」をオフにする。

**Ollama利用時の代替手法（参考）：**
```modelfile
# qwen35_nothink.modelfile
FROM qwen3.5:9b
TEMPLATE """...<ユーザー入力> /no_think<|im_end|>..."""
```
→ `ollama create qwen35-nothink -f qwen35_nothink.modelfile`

---

## Ollama の `launch` コマンド（補足）

Ollama 0.20.7以降は `ollama launch` でAIツールをローカルモデル経由で起動できる。
Aider連携では不要だが、Claude Code をローカルLLMで動かしたい場合に有用。

```bash
ollama launch claude --model qwen3.5:9b
```

→ 詳細は [[reference_ollama_launch]] 参照（メモリファイル）

---

## ROG Flow Z13 での運用設定

- 動作モード：**Performance（CPU 45–50W）**
- USB-C 100W PD給電（LLM推論時はバッテリー減りが速い）
- iGPU VRAM割り当て：**8GB**（MLX Qwen3-14B 4bit なら足りる）
- LLM推論中はファン：3,000–3,500rpm 目安

→ 詳細は [[FlowZ13_初期セットアップ_最適化チェックリスト]] 参照

---

#localLLM #aider #lmstudio #qwen #環境構築
