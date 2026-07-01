# Amazon注文メールの日付を「お届け日」から取っていた問題

## ① 何をしたかったか

家計簿アプリで、Amazon注文確認メールを自動解析して支出を登録したかった。
メール本文から注文日・金額・商品名を取得する処理を実装。

## ② 何が起きたか

最初の実装で注文日を「7月10日にお届け」という行から取得していた。

```python
m = re.search(r'(\d{1,2})月(\d{1,2})日にお届け', body)
```

これはお届け（配送予定）日であり、注文日ではない。
注文した日と登録される日付がずれてしまう。

## ③ どう解決したか

メールの `Date` ヘッダーから受信日を取得して注文日として使うよう変更した。

```python
from email.utils import parsedate_to_datetime
mail_date = parsedate_to_datetime(msg.get("Date", ""))
received_date = mail_date.strftime("%Y-%m-%d")
register_amazon_order(body, subject, received_date)
```

`register_amazon_order()` に `received_date` 引数を追加し、
渡された場合は本文から推定した日付を上書きする形にした。

## 教訓

メール本文の日付は「注文日」とは限らない。
ヘッダーの `Date` が一番確実な受信・送信日時の情報源。
