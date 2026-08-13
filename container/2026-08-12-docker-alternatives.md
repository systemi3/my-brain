# Docker Desktop以外のコンテナ環境（Colima を選んだ理由）

## ① 何をしたかったか

AIに書かせたシェルスクリプトを、安全に検証できる使い捨て環境が欲しかった。
Docker Desktop は常駐してメモリを食うため、24GBのAirでは他の作業を圧迫する。

## ② 何が分かったか

### コンテナ環境は Docker Desktop だけではない

```bash
# 何が入っているか確認するコマンド
which docker
ls -d /Applications/Docker.app 2>/dev/null && echo "Docker Desktop あり"
which colima orbstack podman 2>/dev/null
```

| 実装 | 特徴 |
|---|---|
| Docker Desktop | 定番。GUIあり。常駐する。企業の業務利用は一定規模以上で有償 |
| **Colima** | MITライセンス。GUIなし。`stop` でメモリを完全に返す |
| OrbStack | 高速・省メモリを謳う。有償プランあり |
| Podman | デーモンレス。RedHat系 |

### Colima の名前の由来 = 構造そのもの

```
Co ntainers
Li nux
Ma c
```

```
Mac の上に  →  Linux(VM)を建てて  →  その中でコンテナを動かす
```

`colima start` の40秒は **Linuxを1台起動していた時間**。
`docker run` が一瞬なのは、その中に部屋を作るだけだから。

```
colima start  = 建物を建てる（重い・数十秒）
docker run    = 部屋を作る（軽い・秒）
```

### 実測：常駐の代償

```
Colima ON  : 21 tok/s
Colima OFF : 30.9 tok/s
```

**ローカルLLMの速度に直接響いた。** 24GB環境では常駐型か停止型かが体感差になる。

### なぜLinux上が「本場」なのか

コンテナ技術の正体は **Linuxカーネルの機能そのもの**。
Mac/Windowsにはカーネルがないので、裏で小さなLinux VMを動かしている。
この1層があるぶん、メモリ確保やファイル読み書きが遅くなる。

```
Mac      : macOS   →  Colima(Linux VM)  →  Docker  →  コンテナ
Windows  : Windows →  WSL2(Linux VM)    →  Docker  →  コンテナ
Linux    : Linux   →                       Docker  →  コンテナ  ← 層がない
```

**Colima は Windows では使えない**（macOS専用）。Windowsでの対応物は WSL2。
ただし3層構造も `docker` コマンドも Dockerfile も全部同じ。

## ③ どう判断したか

### 停止型（Colima）を選んだ理由

用途がこれだから：

```
AIにスクリプトを書かせる → コンテナで実行 → 結果を見る → 捨てる
```

**最初から最後まで使い捨て。常駐する必要が1ミリもない。**

### 停止型のデメリット（理解した上で受け入れる）

- 常駐サービス（Difyなど）が持てない
- `restart=always` が効かない（Colimaが止まれば復帰の土台がない）
- 起動直後はディスクキャッシュが空で遅い
- **ただしデータ（ボリューム）は stop しても消えない**

### 役割分担の地図

| 用途 | マシン |
|---|---|
| 常駐サーバー（Dify・開発用DB） | Mac mini（48GB） |
| 使い捨て検証 | Air（24GB）＝ 停止型のまま |

LM Studio でも同じ結論だった——**Airは一度に一つ、Mac miniは同時に複数**。

## 教訓

- **「入っている」と「動く」は別。** `docker --version` が通っても `docker info` で
  `Cannot connect to the Docker daemon` なら、CLIはあるが本体が止まっている
- これは aider/LM Studio と全く同じ「クライアントとサーバー」の構造
- 動いているものを触らない（`--vm-type vz` への変更などは必要になってから）
