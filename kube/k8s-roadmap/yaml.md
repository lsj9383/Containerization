# YAML

[TOC]

## 概览

本文主要参考：[YAML tutorial: Get started in 5 minutes](https://www.educative.io/blog/yaml-tutorial?aff=KNLz)。

什么是 YAML？

> YAML is a data serialization language for storing information in a human-readable form.

YAML 早期的缩写是："Yet Another Markup Language"，后面把含义修改成了： "YAML Ain't Markup Language"，以区别标记性语言。

YAML 类似于 XML 和 JSON，，但使用更简约的语法，更方便人类可读，同时保持类似的功能。

## YAML vs JSON vs XML

序列化语言 | 使用场景 | 可读性 | 语法 | 注释支持 | 所需容量
-|-|-|-|-|-
YAML | 需要开发人员共享或者频繁阅读的数据文件，例如配置信息。| 高 | 简 | 支持 | 低
JSON | 通信使用。 | 中 | 严 | 不支持 | 中
XML | 需要更丰富的控制性。 | 低 | 严 | 支持 | 高

## YAML 显著特征

以下是 YAML 提供的一些实用功能。

### 多文档支持

在一个 YAML 文件中，可以存在多个 YAML 文档（document），不同的 YAML 文档通过 `---` 进行分割：

```yaml
---
player: playerOne
action: attack (miss)
---
player: playerTwo
action: attack (hit)
---
```

### 内建注释

YAML 支持使用 `#` 进行注释：

```yaml
key: #Here is a single-line comment 
   - value line 5
   #Here is a 
   #multi-line comment
 - value line 13
```

### 更可读的语法

相比于 JSON，减少了大括号、方括号、引号等噪音：

```yaml
Imaro:
   author: Charles R. Saunders
   language: English
   publication-year: 1981
   pages: 224
```

一个等价的 JSON：

```json
{
  "Imaro": {
    "author": "Charles R. Saunders",
    "language": "English",
    "publication-year": "1981",
    "pages": 224,
  }
}
```

### 显式转换

YAML 支持显示指定数据类型，只需要在值前使用 `!![typeName]` 即可：

```yaml
# The value should be an int:
is-an-int: !!int 14.10
# Turn any value to a string:
is-a-str: !!str 67.43
# The next value should be a boolean:
is-a-bool: !!bool true
```

### 无可执行命令

作为纯数据格式，YAML 不支持包含可执行文件，因此使用 YAML非常安全。

## YAML 语法


