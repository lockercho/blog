---
title: Python 尬設計模式：沒有 Interface，但有 abc
date: 2020-06-17 22:07:34
tags: [python, oop, design-pattern]
---
本篇文章用來記錄我在工作上寫 Python 時使用 Design Pattern 的一些心得。

Python 不是一個典型的 OOP 語言（相對於 Java 而言），要用 Python 寫 OOP 時難免有些卡卡的。如果你跟我一樣先從別的語言（Java, C++, ...）學過 Design Pattern 了，現在想把知識拉到 Python 用卻不知道從何下手，這篇文章可能會對你有些幫助。

（另外，本文預設讀者已經懂 OOP、Design Pattern、Interface 與 Python）
（如果沒有先行知識，這篇文章可能會讓你看得很混淆，慎入！）


## Python 沒有 Interface

Design Pattern 經常需要使用 Interface 來達成多型。Python 原生沒有 Interface，但還是有些方法可以做出 Interface。

### Option 1: Interface 是啥？我有多重繼承！
Python 可以多重繼承，基本上可以寫一堆 class 叫做 XxxInterface，然後多重繼承這些 class 就好。不過這樣的寫法會重用 Super class 的實作，跟 Interface 的主功能：定義需要的 property/method/input/output 而不實作，還是有一點點差異。

### Option 2: abc (Abstract Base Class)
要更 fancy 的話， 用 abc 這個 ~名字有點鬧的~ module 可以寫出更接近 "Interface" 的東西。

要定義一個 Interface，你需要寫一個 class 繼承 `abc.ABC`，並把 `abstract method` 跟 `abstract property` decorate 好，這就是你的 Interface。要實作這個 Interface 時用繼承的就好。

以下是範例：
```
import abc

# 定義 Interface

class BookInterface(abc.ABC):
    # 定義 read-only property
    @abstractproperty
    def author(self):
        return self.__author

    # 定義 read-write abstract property
    def getp(self):
        return self.__price

    def setp(self, value):
        self.__price = value

    price = abc.abstractproperty(getp, setp)

    # 定義 abstract method
    @abc.abstractmethod
    def read(self):
        print("Interface Impl")


# 繼承（實作） BookInterface

class AmazonEbook(BookInterface):
    @property
    def price(self):
        return self.__price

    @price.setter
    def price(self, value):
        self.__price = value

    def read(self):
        print("AmazonEbook read")
```
上面抄走就可以開始寫 Interface 囉！

\OOP/ \Design Pattern/ \Polymorphism/ 尬起來！

#### Abstract Base Class 提供的好處
如前面所說，多重繼承其實就可以處理基本的 Interface 問題了，而相對於用多重繼承硬尬，ABC 還多提供了以下輔助：
1. 如果少 implement 了 abstract method 或 abstract property，Object initialization 時會報 `TypeError`。

2. 使用 abc 可以 overload `__subclasscheck__`, `__instancecheck__`，讓你有機會改變 `issubclass()` 跟  `isinstance()` 的預設行為。

另外還有些好處可能跟 extend / implement 一些 built-in Type 如 Sets, Sequences 有關，但這部分我不熟所以就不多做介紹以免誤導，有興趣的人可以看看 [PEP 3119](https://www.python.org/dev/peps/pep-3119/)。

#### Abstract Base Class 依舊不是 Interface
拿 abc 當 Interface 用，寫一寫可能還是會發現哪裡怪怪的。
這邊列了一些從 Interface 轉來的人會覺得 abc
1. 如果少 implement 了什麼 method，會到 init object 才噴錯。這算是 Python 的原罪，可以使用 Pylint 之類的 linter 自動找出沒 implement 的 method 來處理掉這個問題。
2. abstract method **_可以_** 有 concrete implementation，並能用 super() 取用：
```
class SimpleInterface(abc.ABC):
    @abc.abstractmethod
    def method_a(self):
        print("I'm from an Interface.")

class SimpleConcrete(SimpleInterface):
    def method_a(self):
        super().method_a()

SimpleConcrete().method_a() # 會印出 "I'm from an Interface."
```
3. Implement abstract method 時，arguments 名稱跟數量都可以不一樣。（但有的 linter 會提示你請取一樣的名稱）

4. 甚至也 **_可以_** 用 property implement method 或用 method implement property。
（畢竟 Python 是 first class function，大家都是 object）
（只是 call 錯時還是會噴錯就是了XD）


## 但我聽說 Python 是 Duck Typing 所以根本不用 Interface？

[Duck Typing](https://en.wikipedia.org/wiki/Duck_typing)（鴨子型別）的意思是：「當看到一隻鳥走起來像鴨子、游泳起來像鴨子、叫起來也像鴨子，那麼這隻鳥就可以被稱為鴨子。」
也就是說一個 class 不管他實際上繼承自哪，只要有相應的 method/property，Python 就允許這個 class 到處 ~招搖撞騙~ 被當作其他特定 class 使用。

abc 的作者也在 PEP-3119 有一段 [ABCs v.s Duck Typing 的討論](https://www.python.org/dev/peps/pep-3119/#abcs-vs-duck-typing)。大意是說 abc 不是拿來殺害 Duck Typing 的。很多時候只要使用 `hasattr()` 來判斷需要的 attribute 存在就已經寫的 666 了，那就不一定要使用 abc。

其實我自己覺得 Python 的 Duck Typing 跟多重繼承，合起來是 self-contained 的一套了。至少在 2007 年以前 abc 都不存在，與其尋找類 Interface 的替代品，不如深入了解 Python 語言本身的強項/限制/理念，來找出情境下最合適的寫法。雖然我在上面列了一些 abc 會讓人感到「怪怪的」點，但再稍微思考一下，那些點都不是 abc 故意設計的，而是 Python 本身特性自然造就的而已。

就算到今日 abc 已經很成熟，何時使用 abc、如何使用也都該交由使用情境來決定，總之就是沒有銀彈。


最後推薦閱讀本文不斷出現的 [PEP-3119](https://www.python.org/dev/peps/pep-3119/)，這是當初 Python 社群提出 abc 的 proposal。裡面詳細寫了為什麼要有 abc、相關用法以及跟其他 alternatives 的比較討論。