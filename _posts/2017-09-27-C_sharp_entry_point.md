---
title: Доступные сигнатуры метода Main и что выбрать
tags: .NET C#
redirect_from: "/C_sharp_entry_point/"
---

Опираясь на исходный код метода [HasEntryPointSignature](https://github.com/dotnet/roslyn/blob/3bbb684a43a0af9d1261866272274a19f4de6976/src/Compilers/CSharp/Portable/Compilation/CSharpCompilation.cs#L1637), который используется компилятором для нахождения точки входа в наше приложение, можно смело заключить, что:

- сигнатура метода должна определять возвращаемое значение как `int` или `void`;
- сигнатура метода должна определять входной аргумент как `string[]` или его отсутствие;

В противном случае [Compiler Error CS5001](https://docs.microsoft.com/ru-ru/dotnet/csharp/misc/cs5001) нарочито сообщит нам о допущенной ошибке:

> Main must be declared as static and it must return void or int, and it must have either no parameters or else one parameter of type string[].

**Начиная с C# 7.1** так же доступно использование `async` keyword и возвращаемые значения типов `Task` и `Task<int>`. Для успешной компиляции необходимо используя Visual Studio 15.3 редакции в `Project properties > Build > Advanced > Language version` выставить значение `C# 7.1`:

![chsarp_71_ep](/assets/images/posts/chsarp_71_ep.gif)

Выбор необходимой сигнатуры обуславливается необходимостью возвращать результат выполнения нашего приложения и конфигурации посредством передаваемых аргументов.

Однако, наличие более одного класса содержащего метод `Main` не воспрещается и определение точки входа может осуществляться на этапе компиляции с использованием аргумента [/main](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/main-compiler-option).
