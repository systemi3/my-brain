# Mac mini M4 Pro 48GB セットアップ：順番の決め方と手順

新しいマシンに開発環境を作るとき、**何をどの順番でやるか**をどう決めたかの記録。
コマンドそのものより、順番を決めた理由が後で効く。

## ① 何をしたかったか

Mac mini M4 Pro 48GB が届くにあたり、以下を構築したい。

- ローカルLLMサーバー（3台どこからでも使える相棒）
- 電車アプリのデータ収集サーバー（24時間稼働）
- 開発環境（Next.js / Python / Docker）

## ② 順番をどう決めたか

### 原則1：依存関係の下から積む

```
基盤（Homebrew/Git）→ ネットワーク → 用途別ツール
```

**なぜ：** 下から積めば、途中で詰まっても前の段が無駄にならない。
逆に上から作ると、土台を入れ直すたびに上が壊れる。

### 原則2：「止まると欠測するもの」を最優先にする

当初の計画ではローカルLLMが1週目だったが、**データ収集を先に繰り上げた。**

| タスク | 遅れた場合 |
|---|---|
| データ収集の常駐化 | **止まった分がそのまま欠測。取り返せない** |
| ローカルLLM構築 | 1週間遅れても失うものがない |

> **「後から取り返せるか」で優先度を決める。**
> 時間に敏感なのは、遅れが不可逆な損失になるタスク。

### 原則3：到着日に欲張らない

初日は「基盤・実測・ネットワーク」の3つだけ。
**欲張るとセットアップ作業に追われて、本来の目的（習慣づくり）が後回しになる。**

### 最終的な時間割

| 時期 | やること |
|---|---|
| 初日 | Homebrew / Git / SMART測定 / Tailscale / スリープ無効化 |
| 翌日〜2日目 | データ収集の常駐化（launchd） |
| 1週目 | ローカルLLM（LM Studio / Ollama） |
| 2週目 | 開発環境の残り（Node / uv / Claude Code / Docker） |

## ③ 各ステップの要点

### 基盤

```bash
xcode-select --install

/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# ★インストール後の「Next steps」2行を必ず実行（忘れるとbrewが見つからない）

git config --global user.name "systemi3"
git config --global user.email "GitHubに登録しているメール"
git config --global init.defaultBranch main
```

### 実測ベースライン（初日にしかできない）

SMART測定でSSDの初期値を記録する。
**「新品時の数値」は今しか取れない。** 後から劣化を測るための基準点になる。

### ネットワーク（Tailscale）

```bash
brew install --cask tailscale-app
open -a Tailscale
# → 他の端末と同じアカウントでログイン。既存ネットに自動参加

tailscale ip -4     # 100.x.x.x をメモ。今後ずっと使う
```

Air側から `ping 100.x.x.x` で疎通確認。

### サーバー化の要：スリープさせない

システム設定 → エネルギー →
- 「ディスプレイがオフのときに自動でスリープさせない」→ オン
- 「ネットワークアクセスによるスリープ解除」→ オン

**スリープすると「24時間そばにいる相棒」になれない。** サーバー機の大前提。

### リモート操作

システム設定 → 一般 → 共有 → 「画面共有」「リモートログイン(SSH)」をオン
→ `ssh ユーザー名@100.x.x.x` でディスプレイなし運用が可能に

### 開発環境

```bash
brew install node
brew install uv                                    # Python環境
curl -fsSL https://claude.ai/install.sh | bash     # Claude Code（ネイティブ版）
brew install --cask docker-desktop
brew install supabase/tap/supabase
```

**なぜ uv か：** pyenv や venv の手動管理より速く、
プロジェクトごとに `uv sync` 一発で環境が揃うから。

**なぜ Claude Code はネイティブ版か：** Node.js不要で、
バックグラウンド自動更新される（npm版は手動更新）。診断は `claude doctor`。

### 環境変数の移行（手作業が必要）

`.env.local` は .gitignore 対象なので **cloneでは来ない。手で運ぶ。**

```bash
scp ~/Documents/プロジェクト/.env.local ユーザー名@100.x.x.x:~/Documents/dev/プロジェクト/
```

## ④ 移行時の型：並行稼働してから切り替える

**収集サーバーを引っ越すとき、いきなり切り替えない。**

1. 新環境（Mac mini）側を起動
2. **丸1日、旧環境（iMac/Air）も止めずに両方動かす**
3. 翌日、両方の出力行数を比較

```bash
wc -l data/jre_$(date +%Y%m%d).jsonl
```

4. 新環境が同等以上なら、旧環境を停止

**なぜ並行させるか：**
「新環境が動いたつもりで実は動いていなかった」場合、
切り替えた瞬間から欠測が始まる。**確認手段は1つに頼らない。**

## ⑤ 詰まりやすいポイント（実体験）

| 症状 | 原因と対処 |
|---|---|
| `brew` コマンドが見つからない | Homebrewインストール後の「Next steps」2行の実行忘れ。`eval "$(/opt/homebrew/bin/brew shellenv)"` |
| devサーバーが二重起動 | `kill -9 [PID]` してから `npm run dev` |
| `claude` コマンドが見つからない | ターミナルの開き直し（PATH反映） |
| Tailscaleで届かない | 全端末が同じアカウントでログインしているか確認 |

## 教訓

- **順番を決める基準は「依存関係」と「取り返しがつくか」の2つ。**
  機能の重要度ではない
- **初日にしか取れないデータ（新品時のSMART値）がある。** 実測主義なら初日に取る
- **移行は「並行稼働 → 比較 → 切り替え」。** 一発切り替えは欠測のリスク
- 役割分担を明確にする：Air＝Web開発、Mac mini＝収集・分析サーバー
