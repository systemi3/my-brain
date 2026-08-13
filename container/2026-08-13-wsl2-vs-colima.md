# 【番外】Windows の WSL2 と Colima の違い

Mac で Colima を選んだ判断が、Windows でそのまま通用するかを整理したメモ。

## ① 何を知りたかったか

Mac では「使うときだけ `colima start`、終わったら `colima stop` でメモリが完全に返る」
という停止型の運用を選んだ。Windows で同じことができるのか？

## ② 分かったこと

### 構造（3層）は完全に同じ

```
Mac      : macOS   →  Colima(Linux VM)  →  Docker  →  コンテナ
Windows  : Windows →  WSL2(Linux VM)    →  Docker  →  コンテナ
                        ↑ ここが入れ替わるだけ
```

`docker` コマンドも Dockerfile も docker-compose.yml も全部同じ。
**学んだことは無駄にならない。**

### 決定的な違い：メモリの返り方

| | Colima (Mac) | WSL2 (Windows) |
|---|---|---|
| 設計思想 | **停止型**。使うときだけ起動 | **常駐型**。Windows起動中はほぼ動いている |
| メモリ解放 | `colima stop` で1バイトも残らず返る | 放っておくと**掴んだまま返さない**（vmmem/VmmemWSL問題） |
| 上限設定 | `colima start --memory 2` | `.wslconfig` に書いて `wsl --shutdown` |
| 設定の形 | コマンド1行 | 設定ファイル |

**WSL2はデフォルトで全RAMの最大50%を掴む可能性がある。** これが
「Dockerがメモリを食い尽くす」と言われる正体。

### WSL2でメモリを返させる設定

`C:\Users\ユーザー名\.wslconfig` を作る（拡張子なし・要注意）。

```ini
[wsl2]
memory=8GB          # 上限を明示。Windows本体に最低4GBは残す
processors=4
swap=2GB

[experimental]
autoMemoryReclaim=gradual   # アイドル時に少しずつメモリを返す
```

反映するには Windows 側（WSL内ではない）で：

```powershell
wsl --list --running    # 何が動いているか
wsl --shutdown          # 再起動（全コンテナが止まる）
```

確認は WSL 内で `free -h`。設定値と一致していればOK。

### ⚠️ 落とし穴（調べて分かった注意点）

1. **`autoMemoryReclaim` は Windows 11 + WSL 2.0.0 以降が必要。**
   Windows 10 では `[experimental]` ごと不要（memory上限だけでも効果あり）
2. **`autoMemoryReclaim` は WSL内で dockerd を直接動かしていると壊すことがある。**
   Docker Desktop 経由なら影響なし。Docker が変にハングしたらここを疑う
3. **`.wslconfig` はメモ帳が勝手に `.txt` を付けると無言で無視される。**
   エラーも出ないので「設定したのに効かない」の原因No.1
4. **実験的機能なのでキー名が変わることがある。** 古い記事の設定名が
   通らないときは、認識されないキーは黙って無視されるだけ

## ③ どう理解したか

### 選択の思想が逆

```
Colima : 「使うときだけ建物を建てる」  → 明示的に start / stop
WSL2   : 「建物は建てっぱなし、部屋の広さを制限する」 → 設定ファイルで上限を決める
```

**Colimaは"電源スイッチ"で管理、WSL2は"上限設定"で管理。**
だから WSL2 では `.wslconfig` を書かないと際限なく膨らむ。

### つまり

- **Colima は Windows では使えない**（macOS専用。名前が Containers on Linux on **Ma**c）
- 対応物は WSL2 だが、**「終わったら止めてメモリを返す」という発想がそのままは通用しない**
- Windows でやるなら「`.wslconfig` で上限を決めて、必要なら `wsl --shutdown`」が近い運用

### Linuxが最速な理由（再確認）

コンテナの正体は Linux カーネルの機能。Mac も Windows も
**Linux VM という1層を挟んでいる**から遅い。Linux 上ではその層がない。

```
Linux : Linux → Docker → コンテナ   ← VM層がない
```

## 教訓

- **道具が変わっても3層構造は同じ。** 構造で覚えておけば、環境が変わっても迷わない
- ただし**「同じ構造」と「同じ運用」は別物**。設計思想（停止型 / 常駐型）まで
  同じとは限らない
- 設定ファイルが「無言で無視される」タイプの落とし穴は、
  エラーが出ないぶん一番時間を溶かす
