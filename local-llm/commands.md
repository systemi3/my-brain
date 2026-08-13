# ローカルLLM コマンド早見表

困ったときに最初に見る場所。上から順に試せば大抵解決する。

---

## 🚨 生成が止まらないとき（最優先）

**Mac mini（LM Studioがある側）で実行する。** aider側で何をしても止まらない。

```bash
lms ps                  # 今なにが動いているか確認
lms unload --all        # 全モデルをアンロード = 生成も強制終了
```

`lms` が見つからない場合は `~/.lmstudio/bin/lms`。
GUIが触れるなら右下の **Eject** ボタンでも同じ。

それでも変なときだけサーバーごと再起動（通常は不要）：

```bash
lms server stop
lms server start
```

---

## 接続確認（トラブル時に必ず最初に試す）

```bash
# ローカルの場合
curl -s http://127.0.0.1:1234/v1/models | python3 -m json.tool

# LAN/Tailscale越しの場合
curl http://[Tailscale IP]:1234/v1/models
```

JSONが返ればサーバーは生きている。
返らなければ LM Studio の Developer タブでサーバーを Start する。

---

## LM Studio セットアップ

```bash
brew install --cask lm-studio
open -a "LM Studio"
```

LAN公開：Developerタブ →「Serve on Local Network」オン → Running

**MLX版を選ぶ理由：** Apple Silicon専用フォーマットでGGUFより速い。

---

## Ollama（API呼び出し用）

```bash
brew install ollama
brew services start ollama
ollama pull qwen2.5:14b
ollama run qwen2.5:14b "こんにちは"
```

**役割分担：** LM Studio＝対話・実験 / Ollama＝API呼び出し

---

## aider の設定ファイル（3つ）

すべて **aiderがある端末（Air）側** に置く。これで警告が消え、暴走しにくくなる。

| ファイル | 役割 |
|---|---|
| `~/.aider.model.metadata.json` | モデルのスペック（コンテキスト長・コスト）を教える |
| `~/.aider.model.settings.yml` | 振る舞い（編集形式・max_tokens）を決める |
| `~/.aider.conf.yml` | 接続先を固定し、毎回の環境変数を不要にする |

**設定前後の起動コマンドの差：**

```bash
# Before
OPENAI_API_BASE=http://100.78.4.110:1234/v1 OPENAI_API_KEY=lm-studio \
  aider --model openai/qwen3.6-35b-a3b-mlx

# After
aider
```

### alias（設定ファイルを使わない場合）

```bash
alias aider1="OPENAI_API_BASE=http://127.0.0.1:1234/v1 \
              OPENAI_API_KEY=lm-studio \
              aider --model openai/qwen2.5-coder-14b-instruct"
```

| 変数 | 意味 |
|---|---|
| `OPENAI_API_BASE` | LM Studio サーバーのアドレス |
| `OPENAI_API_KEY` | `lm-studio` という固定文字（ローカルなので認証不要だが必須） |
| `--model` | LM Studio で選択中のモデルと一致させる |

---

## aider の中で使うコマンド

| コマンド | 用途 |
|---|---|
| `/ask 質問` | 編集せずに聞くだけ（安全確認に使う） |
| `/undo` | 直前の変更を取り消す（gitの履歴が戻る） |
| `/drop` | 追加ファイルを全部外す |
| `/add ファイル` | ファイルを対象に追加 |
| `/clear` | 会話履歴をリセット |
| `Ctrl+C` | 中断（安全） |

**新しいファイルを作らせたいとき：** aiderは `/add` されていないファイルに書けない。

```bash
touch src/lib/newfile.ts
# aider内で
/add src/lib/newfile.ts
```

---

## モデル選定の目安

| メモリ | 現実解 | できること |
|---|---|---|
| 8〜12GB | Qwen2.5-Coder 7B | 単発スクリプト生成 |
| 16GB | Qwen2.5-Coder 14B / Phi-4 14B | 単体作業なら十分 |
| 24GB〜 | Qwen3-Coder 30B-A3B (Q4) | 調査〜作成〜デバッグまで安定 |

**大きいモデル＝正確ではない。** 強い量子化が必要になり劣化する。
→ 詳細: [機材選定の調査](2026-08-13-model-spec-selection.md)

### ログ解析の定石

巨大ログを丸ごと読ませない。**grep等で絞ってから要約させる。**
コンテキストが長いほど速度と精度が落ちるため。

```bash
grep "ORA-" alert.log | tail -100 > /tmp/subset.log
```

### メモリ運用のコツ

常時30Bを起動しっぱなしにしない。
「普段は9Bで会話、重い解析のときだけ30Bを起動」で余裕が出る。

```bash
lms ps              # 今なにが載っているか
lms unload --all    # 全部降ろす
```

---

## 読み方（間違えやすい）

| 表記 | 読み |
|---|---|
| **Aider** | **エイダー**（アイダーではない） |
| **Qwen** | **クウェン**（アリババ製） |
| Colima | コリマ |
| Ollama | オラマ／オーラマ |
| Podman | ポッドマン |
| MLX | エムエルエックス |
