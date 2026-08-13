# コンテナ コマンド早見表（Colima / Docker）

---

## 🔍 まず状態を確認する

```bash
docker --version              # コマンドが存在するか（動くかは分からない）
docker info 2>&1 | head -20   # 実際に動くか
```

| 結果 | 状態 | 対処 |
|---|---|---|
| バージョンが出て info も正常 | ✅ すぐ使える | — |
| `Cannot connect to the Docker daemon` | ⚠️ CLIはあるが本体が未起動 | `colima start` または `open -a Docker` |
| `command not found` | ❌ 入っていない | インストールから |

**なぜ2つ打つか：** `--version` は「コマンドがあるか」しか答えない。
**「入っている＝動く」ではない。**

### 何が入っているか特定する

```bash
which docker
ls -d /Applications/Docker.app 2>/dev/null && echo "Docker Desktop あり"
which colima orbstack podman 2>/dev/null
```

---

## Colima（コリマ）

```bash
colima start                       # 起動（設定を記憶しているので2回目以降はこれだけ）
colima start --cpu 2 --memory 2    # 初回・設定変更時。この1行が設定そのもの
colima stop                        # 停止。メモリが完全に返る
colima list                        # 状態確認
```

### Docker Desktop 前提の記事を読むときの読み替え表

```
Settings → Resources → Memory   →  colima start --memory N
アプリを起動                     →  colima start
アプリを終了                     →  colima stop
```

### メモリ確認

```bash
docker info | grep -i "total memory"
sysctl vm.swapusage
memory_pressure | tail -3
```

---

## Docker 基本

```bash
docker run hello-world             # 動作確認の定番
docker ps                          # 動いているコンテナ
docker ps -a                       # 停止中も含む全コンテナ
docker images                      # イメージ一覧
docker volume ls                   # ボリューム一覧（★データはここ）
```

**`docker volume ls` が重要：** イメージが消えていても
ボリュームが残っていればデータは復活する。

### 検証用コンテナに入る

```bash
docker run -it -v ~/ai-sandbox:/work ubuntu:24.04 bash

# コンテナ内で
cd /work
ls
cat /etc/os-release
exit
```

---

## 🧹 掃除の習慣（月1回）

```bash
docker system df       # 何にどれだけ使っているか
docker system prune    # 使っていないものを削除
```

**なぜ習慣化するか：** イメージとコンテナは黙って溜まる。
月1で `df` を見る癖をつけると、数十GBを失う事故を防げる。

---

## Docker Compose（複数サービス）

```bash
docker compose up -d      # 起動（バックグラウンド）
docker compose down       # 停止。★データ（ボリューム）は残る
docker compose logs -f    # ログを見る
```

### docker-compose.yml の読み解きポイント

```yaml
services:                 # ← 複数のコンテナを並べる
  db:
    image: mariadb:11
    volumes:
      - db_data:/var/lib/mysql    # ← これがないと down で全消去
  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"                 # ← Mac の8080 → コンテナの80
    depends_on:
      - db

volumes:                  # ← 永続化するデータの宣言
  db_data:
```

| 記述 | 意味 |
|---|---|
| `services:` | サービス（コンテナ）ごとの定義 |
| `volumes:` | コンテナを消してもデータが残る |
| `ports: "8080:80"` | ホストの8080番の扉をコンテナの80番につなぐ |
| `depends_on:` | 起動順序の依存関係 |

**ポート番号は他と衝突しないように。**（例：Supabaseは54321、LM Studioは1234）

---

## ローカル Supabase

```bash
supabase start      # 約10個のコンテナが起動（初回はDLで数分）
supabase stop       # 終了。使わない時は止める（2〜4GB消費）
supabase db reset   # ★ローカルでは push ではなく reset
                    #   全マイグレーションをまっさらなDBに順番に適用
                    #   = マイグレーションの検証も兼ねる
```

| URL | 用途 |
|---|---|
| http://localhost:54321 | API |
| http://localhost:54323 | Studio（管理画面） |
| http://localhost:54324 | Inbucket（疑似受信箱。実メールは飛ばない） |

Next.js側は環境変数2行の書き換えだけで切り替わる：

```
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=（startで表示されたキー）
```

→ 詳細: [ローカルSupabaseとセルフホスト](2026-07-04-local-supabase-selfhost.md)

---

## 【番外】Windows / WSL2 のコマンド

Colima は macOS 専用。Windows での対応物は WSL2。

```powershell
wsl --version           # バージョン確認
wsl --list --verbose    # ディストリと状態
wsl --list --running    # 動いているものだけ
wsl --shutdown          # 再起動（=設定反映。全コンテナが止まる）
```

設定は `C:\Users\ユーザー名\.wslconfig`（拡張子なし）：

```ini
[wsl2]
memory=8GB
processors=4
swap=2GB

[experimental]
autoMemoryReclaim=gradual
```

**確認：** WSL内で `free -h`。設定値と一致していればOK。

⚠️ メモ帳が `.txt` を付けると**無言で無視される**（エラーも出ない）。
⚠️ `autoMemoryReclaim` は Windows 11 + WSL 2.0.0 以降が必要。

→ 詳細: [WSL2 と Colima の違い](2026-08-13-wsl2-vs-colima.md)

---

## 運用ルール

- 使い終わったら `colima stop`（Airの場合）／`docker compose down`
- 常駐が必要なもの（Dify等）は Mac mini 側で
- Docker Desktop を使う場合は、終了するとメモリが返る（メニューバーのクジラ → Quit）
