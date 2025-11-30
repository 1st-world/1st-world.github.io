---
layout: post
title:  "Python에서 immutable, mutable 타입"
author: 1st-world
date: 2025-10-09 23:00:00 +0900
categories: [Development, Python]
tags: [type, mutable, immutable]
---

## ✅ 정의:

> **Immutable(불변)**: 값을 한 번 만들면 **바꿀 수 없는** 데이터 타입  
> **Mutable(가변)**: 값을 만든 후에도 **내용을 바꿀 수 있는** 데이터 타입

---

## 📌 예시로 비교

| 타입                   | 예시                       | 변경 가능?            |
| -------------------- | ------------------------ | ----------------- |
| `int` (정수)           | `a = 10`                 | ❌ 불가능 (immutable) |
| `float` (실수)         | `b = 3.14`               | ❌ 불가능             |
| `str` (문자열)          | `s = "hello"`            | ❌ 불가능             |
| `tuple` (튜플)         | `t = (1, 2, 3)`          | ❌ 불가능             |
| `frozenset` (고정된 집합) | `fs = frozenset([1, 2])` | ❌ 불가능             |
| `list` (리스트)         | `l = [1, 2, 3]`          | ✅ 가능 (mutable)    |
| `dict` (딕셔너리)        | `d = {'a': 1}`           | ✅ 가능              |
| `set` (집합)           | `s = {1, 2}`             | ✅ 가능              |

---

## 🔍 예시 코드로 이해하기

### ✅ Immutable (예: `int`)

```python
a = 10
b = a
b = 20

print(a)  # 10
```

* `b = a`를 해도, `b`가 20으로 바뀌면 `a`는 영향 안 받음.
* 이유: `int`는 immutable이므로 **새 값을 할당하면 새로운 객체가 만들어짐**.

---

### ✅ Mutable (예: `list`)

```python
a = [1, 2, 3]
b = a
b.append(4)

print(a)  # [1, 2, 3, 4] ← a도 같이 바뀜
```

* `a`와 `b`는 **같은 리스트를 공유**하고 있어서, 하나를 바꾸면 둘 다 바뀜.
* 리스트는 **mutable**이므로 내용 변경 가능.

---

## 🧠 요약 정리

* **Immutable**: 변경 불가 (`int`, `str`, `tuple`, `float` 등)

  * 새로운 값을 할당하면 **새 객체 생성**.
* **Mutable**: 변경 가능 (`list`, `dict`, `set` 등)

  * 값이 바뀌면 **같은 객체 안에서 바뀜**.

---

> _이 글의 초안은 ChatGPT의 GPT-5 모델과 주고받은 질의응답 내용을 토대로 작성되었습니다._
{: .prompt-info }