---
title: Python 3.9 有哪些新玩意？
tags:
  - python
date: 2020-06-20 15:41:08
---


[Python 3.9](https://docs.python.org/3.9/whatsnew/3.9.html) 即將在今年 10 月初 release，本文介紹一些 Python 3.9 的新東西。

## New Features
### Dictionary Merge & Update Operators
Dictionary 引進新的 operator `|`，可以方便的 merge 與 update dictionary。
以下範例扣比較新舊語法的差異：
```
a = {"a_key": "a_value"}
b = {"b_key": "b_value"}

# Before 3.9
merged = {**a, **b}

# >= 3.9
merged = a | b
```

如果是要把 dict b 的內容 update 進 dict a：
```
# Before 3.9
a.update(b)  # a 的內容被更新，update() return None

# >= 3.9
a |= b
```
可以看到新的語法更加乾淨直覺。

### Typing: Builtin Generic Types [(PEP 585)](https://www.python.org/dev/peps/pep-0585/)
名詞定義：
- Generic 指的是可以被參數化的型別，通常是容器，用 `Dict`, `List` 來想像就可以了。
- Parameterized Generic 就是已指定參數的 Generic，例如 `Dict[str, int]`。

原本在做 typing hint 時，會需要從 typing module 額外 import 大寫的 `Dict`, `Tuple`, `List` 或 `Set` 來宣告你需要的 Parameterized Generic。這是因為 builtin 的小寫 `list`, `dict` 不具備 Generic 的功能（使用 `list[str]` 會報 `TypeError: 'type' object is not subscriptable`。）

在 Python 3.7 後可以藉由 `from __future__ import annotations` 來使用 `list[str]`。

而這次在 Python 3.9 後就是直接原生支援了。
```
# ========== Before 3.7 ========== #

from typing import List

def method() -> List[str]:
    return ["hola"]


# ========== Python 3.7 ========== #

from __future__ import annotations

def method() -> list[str]:
    return ["hola"]


# ========== Python 3.9 ========== #

def method() -> list[str]:
    return ["hola"]

```
這次更新以後小寫的 `tuple`, `list`, `dict`, `set`, `frozenset`, `type` 等等都原生支援 Generic 了，詳細列表可以參考[這裡](https://www.python.org/dev/peps/pep-0585/#implementation)。

我自己原本的用法是，需要 Generic 時才 import 大寫，不需 Generic 時用原生小寫。但時常發 PR 時會被 comment 説請改用大寫的（因為公司沒有明確規範 typing 這邊的 style），不過這版以後全都小寫就好囉。

### String: removeprefix, removesuffix
String 多了兩個小 helper function，可以拿來處理移除前綴後綴：
```
"TestHook".removeprefix("Test")  # => "Hook"
"BaseTestCase".removeprefix("Test") # => "BaseTestCase"

"MiscTests".removesuffix("Tests") # => "Misc"
"TmpDirMixin".removesuffix("Tests") # => "TmpDirMixin"
```

### Decorator Syntax Update [(PEP 614)](https://www.python.org/dev/peps/pep-0614/)
這個改動放寬了 decorator 的語法限制。目前 decorator 的語法裡面不能帶太多的 expression，以下是 PEP 裡面給出的 PyQt 範例：
```
buttons = [QPushButton(f'Button {i}') for i in range(10)]

# Do stuff with the list of buttons...

@buttons[0].clicked.connect
def spam():
    ...
```
以上的語法是不合法的，必須改寫為以下才能動：
```
button_0 = buttons[0]

@button_0.clicked.connect
def spam():
    ...
```

但神奇的是可以使用一些 hack 來達成在 decorator 使用複雜的 expression：
```
# Hack option 1: Identity function:

def _(x):
    return x

@_(buttons[0].clicked.connect)
def spam():
    ...

# Hack option 2: eval:

@eval("buttons[1].clicked.connect")
def eggs():
    ...
```

PEP 614 中有提到，雖然 motivation 範例的需求是 subscripting（i.e. 使用 `[0]` 取用），但與其隨意放寬單一語法，不如直接放寬到允許 `expression`。在這裡的 `expression` 指的是任何可以放在 `if`, `elif` 或 `while` 裡做判斷的東西。

總之以上面的範例來說，以後可以直接寫成這樣了：
```
buttons = [QPushButton(f'Button {i}') for i in range(10)]

@buttons[0].clicked.connect
def spam():
    ...
```

## New / Updated Module
### zoneinfo [(PEP 615)](https://www.python.org/dev/peps/pep-0615/)
zoneinfo 這個 module 在 Python 3.9 被加入了 standard modules 裡面。

以前在台灣工作時其實很少需要處理時區問題，只要把握好 server side 都用 timestamp 或 UTC time，顯示時間、轉換格式都在 client side 處理的原則，幾乎沒有出過問題。

而現在在倫敦的工作有大量的 timezone 問題需要處理。主要原因是英國有日光節約時間，冬季是 GMT+0、夏季是 GMT+1，加上在能源產業，雖然公司是新創公司，但有大量資料要跟業界舊的系統或人工業務對接，導致許多 input data 的時間都是用 local time，同個 time string `13:00` 在夏季跟冬季會使用不同的 timezone offset。最要命的是，在進入冬季的 clock change day 時，時鐘會往回調，所以這天的 local time 會有兩種 `02:00`。

為了處理時區，目前公司用的是 `pytz` 這個 module，未來升級 Python 3.9 時改用 `zoneinfo` 就不用額外安裝了。另外[這裡](https://www.python.org/dev/peps/pep-0615/#using-a-pytz-like-interface) 也討論了為什麼是放 `zoneinfo` 進 standard 而不是 `pytz`。


## New Parser [(PEP 617)](https://www.python.org/dev/peps/pep-0617/)

Python 3.9  把 CPython 的 LL(1) parser 換成 PEG parser。

文件裡寫到 PEG parser 的效能跟原本的 parser 差不多，並且會生成一樣的 AST。這個改動從使用者角度上應該感受不到差異，主要目的是在開發語言的新 feature 時，PEG parser 提供了更多的彈性。而 PEP 617 也提到了本來他們有對 LL(1) parser 做了一些 hack，改用 PEG parser 後就不用繼續 maintain 這些 hack 了。

## 結語
以上是筆者從自己的角度整理出 Python 3.9 比較值得一提的新功能，但當然也可能有些我沒有用到的 feature 對別人來說其實是很重要的改動，所以一樣推薦直接[閱讀原文](https://docs.python.org/3.9/whatsnew/3.9.html)。

（另外因為官方文件還在草稿階段所以會持續更新，建議 release 時可以再回去看一下。）

## Reference
- [官方文件：What’s New In Python 3.9](https://docs.python.org/3.9/whatsnew/3.9.html)
- [PEP 585 -- Type Hinting Generics In Standard Collections](https://www.python.org/dev/peps/pep-0585/)
- [PEP 614 -- Relaxing Grammar Restrictions On Decorators](https://www.python.org/dev/peps/pep-0614/)
- [PEP 615 -- Support for the IANA Time Zone Database in the Standard Library](https://www.python.org/dev/peps/pep-0615/)
- [PEP 617 -- New PEG parser for CPython](https://www.python.org/dev/peps/pep-0617/)
