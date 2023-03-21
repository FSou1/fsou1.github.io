---
title: Отличия string и String
tags: .NET C#
redirect_from: "/String_and_string_difference_in_C-_and_NET/"
---

Буквально сегодня был сбит с толку подобным вопросом, поэтому решил собрать воедино информацию о различиях ключевых слов и соответствующих им по умолчанию типов.

## Определение

String является типом платформы .NET (Framework Class Library (FCL)) пространства имён System.

string является ключевым словом (псевдонимом) языка C#, которому по умолчанию по итогу компиляции соответствует тип System.String.

## Использование

Следствием того, что String является типом сборки System, его использование обязует указание пространства имён (`using System` либо `System.String`).

## Наименования переменных

Т.к. string является ключевым словом, то его использование в качестве имени переменной без спец. символа запрещено:

```csharp
StringBuilder string = new StringBuilder();  // doesn't compile
StringBuilder @string = new StringBuilder(); // compiles
```

В то время как на использование String никаких ограничений не накладывается:

```csharp
StringBuilder String = new StringBuilder();  // compiles
```

## Возможные проблемы при использовании ключевых слов

Ключевому слову long в C# по умолчанию соответствует тип Int64, в то время как в других языках программирования long'у могут соответствовать Int16 или Int32, как например в C++/CLI long'у соответствует Int32 (Richter, CLR Via C#).
