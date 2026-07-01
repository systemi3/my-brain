# ModuleNotFoundError: No module named 'requests'

## ① 何をしたかったか

PythonスクリプトからAPIを叩くために `requests` ライブラリを使おうとした。

```python
import requests
```

## ② 何が起きたか

上記を実行したら、こんなエラーが出た。

```
ModuleNotFoundError: No module named 'requests'
```

## ③ どう解決したか

原因は、仮想環境を有効化せずにスクリプトを実行していたこと。
以下の手順で解決した。

```bash
source venv/bin/activate
pip install requests
```

## 教訓

エラーが出たら、まず「今どの環境（仮想環境）で実行しているか」を疑う。
`pip install` する前に `which python` で確認する癖をつける。
