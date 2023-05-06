---
title: "Base62 (.NET)"
order: 4
header:
  teaser: /assets/images/projects/base62-teaser.png
sidebar:
  - title: "Tech stack"
    text: "C#, .NET, NuGet package"
  - title: "Status"
    text: "Active (2023 - Present)"
  - title: "NuGet"
    text: "[Install](https://www.nuget.org/packages/Base62.Conversion)"
  - title: "Github"
    text: "[Repository](https://github.com/FSou1/Base62.Conversion)"
---

Base conversion is an approach commonly used for URL shorteners. Base conversion helps to convert the same number between its different number representation systems.

The conversion table is `[0-9a-zA-Z]`:

```c#
private const string BASE62 = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
```

The value `999` is converted into `g7` as:

```
999 = 16 * 62^1 + 7 * 62^0 = "g7"
```

where `g` and `7` are the 16th and 7th elements of the `BASE62`.
