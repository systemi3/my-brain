# データを外に出さない：ローカルSupabaseとMac miniセルフホスト

「開発中・計画中のものを、できる限りMac mini内にデータを置きたい」
という要望から、何が可能で何に注意すべきかを整理した記録。

## ① 何をしたかったか

各プロジェクトのデータを、可能な限りローカル（Mac mini内）に保存したい。
※電車アプリは外部API（ODPT）由来なので除外

## ② 分かったこと

### 結論：可能。Supabaseは丸ごとMac mini内でDocker実行できる

Supabaseはオープンソースなので、`supabase start`（実体はDocker Compose）を動かせば
DB・認証・RLSすべてローカルに置ける。既存のマイグレーションファイルもそのまま使える。

| プロジェクト | 判定 |
|---|---|
| 天翔ポータル | ✅ 可能 |
| ローカルLLM関連 | ✅ もともと全部ローカル前提。RAGのpgvectorも同居可 |
| 自作ツール類 | ✅ 可能。Docker学習の実践台として最適 |
| 電車アプリ | ❌ 除外が正解。外部APIなのでローカルはキャッシュ程度 |

### 難易度が変わる分岐点：「自分だけ」か「他人も使う」か

初学者がローカル保存で一番つまずくのは、**作った後に発覚する**次の3点：

1. **可用性** — 入居者が夜中に見る時、Mac miniが寝ていたら（スリープ・停電・再起動）
   サービスは止まる。自分専用なら許容、他人が使うなら要検討
2. **外部公開** — 自分ならTailscaleで十分だが、
   入居者全員にTailscaleを入れてもらうのは非現実的。Cloudflare Tunnel等が必要
3. **バックアップ** — クラウドと違い、消えたら誰も助けてくれない。
   Time Machine＋定期的な `pg_dump` は必須

### 推奨：ハイブリッドから始める

> **開発・検証は全部ローカルSupabase、本番（他人が触る部分）だけクラウド。**
> これなら試行錯誤は無料、データ主権も学べる。

## ③ ローカルSupabase構築の手順

### 仕組み（1分で理解）

`supabase start` を実行すると、裏でDocker Composeが動き、
PostgreSQL・認証(Auth)・API(PostgREST)・管理画面(Studio)など
**約10個のコンテナが一気に起動する。**
＝「クラウドのSupabase一式を、自分のMacの中に丸ごと再現する」

### 手順

```bash
# 1. Docker Desktop をインストール（Supabase CLIはDockerがないと動かない）
brew install --cask docker-desktop

# 2. Supabase CLI
brew install supabase/tap/supabase

# 3. プロジェクトフォルダで初期化（既にlink済みなら不要）
supabase init

# 4. 起動（初回はイメージDLで数分）
supabase start
```

完了すると、接続URL・anonキー・Studio URL（http://localhost:54323）が表示される。

### 既存マイグレーションの適用

**ローカルでは `supabase db push` ではなく `supabase db reset` を使う。**

```bash
supabase db reset
```

これで全マイグレーションがまっさらなDBに順番に適用され、
**「マイグレーションが正しく書けているか」の検証にもなる。**

### Next.js側の切り替えは環境変数2行だけ

```
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=（startで表示されたキー）
```

**なぜ環境変数で切り替えるのか：**
コードを一切変えずに「開発＝ローカル／本番＝クラウド」を行き来できるから。
※LLM呼び出し先を環境変数で切り替える設計と全く同じ思想

### 注意点

- メモリを2〜4GB食うので、使わない時は `supabase stop`
- Authのメールは **Inbucket**（http://localhost:54324）という疑似受信箱に届く。
  実メールは飛ばない

## ④ メモリ予算：DockerとLLMは奪い合う

**Supabaseを動かす＝Dockerを常時起動する**（コンテナが止まる＝DBが止まる）。
その分メモリが減るので、LLMのモデル選択に直結する。

| 用途 | 目安メモリ |
|---|---|
| macOS本体＋常駐アプリ | 約6〜8GB |
| Docker Desktop＋Supabase一式 | 約3〜5GB |
| Qwen2.5-32B（4bit）ロード時 | 約20GB |
| **合計** | **約29〜33GB** |

48GBに対して15GB前後の余裕。**32B級との同居は可能。**

**先に予算表を作る理由：**
後から「LLMが遅い」「Dockerが落ちる」となった時、原因の切り分けができないから。

### 調整すべき2点

1. **Docker Desktopのメモリ上限を設定する**（Settings → Resources → Memory を6GB程度）
   → Dockerは放っておくと空きメモリを積極的に確保しにいくため、
     LLMロード時の取り合いを未然に防ぐ
2. **70B級への欲は封印する** — 4bitでも40GB級を食う。同居構成では非現実的

### 常時起動が必要になるのはいつか

自分しか使わないフェーズなら `supabase start` / `stop` を手動で切り替えればよい。
**常時起動が本当に必要になるのは「他人がいつアクセスするか分からなくなった時」から。**

## ⑤ Docker で置き換えられるクラウドサービス

**Docker Composeを一度覚えると、他のツールも「yaml 1枚→起動」の同じ手順で増やせる。
つまり1つ目の学習コストで残り全部が「ほぼ無料」になる。**

| # | ツール | 用途 |
|---|---|---|
| 1 | **WordPressローカル環境** | 本番を触らずテーマ・プラグイン実験。壊しても `down` でやり直せる |
| 2 | **Gitea（自宅GitHub）** | 「GitHubに置きたくないコード」の置き場 |
| 3 | **Whisper** | 音声をクラウドに送らず文字起こし |
| 4 | **n8n** | Zapier代替。データを外部に通さず連携を組む |
| 5 | **Vaultwarden** | パスワード・APIキーの保管庫を自宅に |

**注意：全部同時に動かすとメモリ予算を侵食する。**
「常時起動はSupabaseだけ、他は使う時だけ起動」が48GB運用の鉄則。

### WordPressローカル環境の例（Compose入門に最適）

```yaml
services:
  db:
    image: mariadb:11
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp
      MYSQL_PASSWORD: wppass
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wp
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp_data:/var/www/html
    depends_on:
      - db

volumes:
  db_data:
  wp_data:
```

```bash
docker compose up -d      # → http://localhost:8080
docker compose down       # 終了（データは残る）
```

**読み解きポイント：**
- `services:` が2つ = WordPress本体とDBは別コンテナ（「複数サービスの連携」の実物）
- `volumes:` があるから、コンテナを消しても記事データが残る。ないと `down` で全消去
- `ports: "8080:80"` = ホストの8080番の扉をコンテナの80番につなぐ
  （Supabaseの54321と番号が違うので衝突しない）

## ⑥ 専用Ubuntu機を建てるべきか → 現時点では「否」

Docker は Linux 上が本場で、速くトラブルも少ない（カーネル機能をそのまま使えるため）。
だが今は不要。理由は3つ：

1. **規模が小さすぎて差が出ない** — 速度差が効くのは本格的な本番運用から
2. **相棒プランと分断される** — LLMはApple Silicon（ユニファイドメモリ＋Metal）が強み。
   Ubuntu機に移してもLLMは持っていけず、機材が2系統に割れて管理コストだけ増える
3. **学ぶことが先に増える** — Linuxインストール・SSH・セキュリティ更新など、
   Docker以前の運用知識が必要。**初学者は「1台で全部」の期間を長く取る方が伸びる**

### 別の問題：CPUアーキテクチャの違い

Mac miniはARM、多くのWindows機やクラウドはx86。
**Mac上でビルドしたイメージがクラウドでそのまま動かないことがある。**
将来クラウド公開する時に `--platform` 指定で解決する。

### 将来の分岐点

入居者が実際に使い始めて「24時間止められない」段階になったら、
その時こそ中古のミニPC＋Ubuntuを常時稼働サーバーに分離し、
Mac miniをLLM専用に戻す——それが自然な進化順。**今は1台で正解。**

## 教訓

- **メモリ予算表を先に作る。** 後から不調が出た時、原因の切り分けができる
- **ローカルは `db push` ではなく `db reset`。**
  マイグレーションが正しく書けているかの検証を兼ねる
- **切り替えは環境変数で。** コードを変えずに開発/本番を行き来できる
- **機材を増やすのは「今の構成で困ってから」。**
  初学者は1台で全部やる期間を長く取る方が伸びる
