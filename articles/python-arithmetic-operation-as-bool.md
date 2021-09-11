---
title: "【Python】bool 型のまま算術演算する"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "programming"]
published: true
---

bool 型は int 型のサブクラスであるため、そのまま算術演算に使用できます。

```python
>>> cnt = 1
>>> cnt += True
>>> cnt
2
```

bool 値は次のように変換されます。

- True -> 1
- False -> 0

```python
>>> int(True)
1
>>> int(False)
0
```

## サブクラスの確認

`issubclass` 関数を使って、関数がサブクラスであるか判別できます。

bool 型は int 型のクラスであることがわかります。

```python
>>> issubclass(bool,int)
True
```

## if 文を使わない例

`flag` という bool 値を使って条件分岐させたい場合、if 文を使わずに算術演算できます。

```python
# if文を使う場合
if flag == True:
  cnt += 1

# if文を使わない場合
cnt += flag
```

## 詳細

もう少し見ていきましょう。

条件式は bool 値が返されます。

```python
>>> 1 == 1
True
>>> 1 == 0
False

>>> type(True)
<class 'bool'>
>>> type(False)
<class 'bool'>
```

bool 値は、int 型とも判定されます。

```python
>>> isinstance(True,bool)
True
>>> isinstance(True,int)
True
```

条件式は、算術演算子より優先順位が低いので、`()` で囲む必要があります。

```python
>>> 1 == 1 + 1 # 1 == (1 + 1)と同じ
False
>>> 1 == (1 + 1)
False
>>> (1 == 1) + 1
2
```

以下はすべて同じ結果になります。

```python
>>> (1 == 1) + 1
2
>>> bool(1 == 1) + 1
2
>>> int(bool(1 == 1)) + 1
2
```

## Reference（参考文献）

https://note.nkmk.me/python-bool-true-false-usage/

https://dev.classmethod.jp/articles/python-isinstance-or-type/

https://www.javadrive.jp/python/num/index3.html

https://docs.python.org/ja/3/c-api/bool.html

https://docs.python.org/ja/3/library/functions.html
