---
title: Перечисление и итераторы (очень кратко, на примере последовательности Фибоначчи)
tags: .NET C#
redirect_from: "/IEnumerable_IEnumerator_and_fibonachi/"
---

Не вдаваясь в пространные рассуждения сегодня хотелось бы очень кратко пояснить одну из самых любимых тем интервьюеров: что нужно сделать, чтобы иметь возможность итерироваться по экземпляру класса конструкцией `foreach`.

Итак, следуя терминологии Джозефа Албахари (C# 6.0 in a nutshell),- _перечеслитель_, это объект, который реализует один из интерфейсов:

```csharp
System.Collections.IEnumerator
System.Collections.Generic.IEnumerator<T>
```

Оператор `foreach` же выполняет итерацию по **перечислимому объекту**, где **перечислимый объект** тот, который:

1. Либо реализует интерфейс `IEnumerable` или `IEnumerable<T>`;
2. Либо имеет метод по имени `GetEnumerator`, который возвращает _перечислитель_ (что на самом деле является единственным необходимым условием, т.к. наличие метода `GetEnumerator` будет и следствием реализации интерфейса `IEnumerable`);

На примере с последовательностью Фибоначчи всё должно стать понятнее.

## Задача

Необходимо вывести только чётные из первых 20 чисел последовательности Фибоначчи.

## Числа Фибоначчи как перечислимый объект

Как мы уже выяснили ранее, чтобы иметь возможность итерироваться по объекту нам нужно соблюсти одно из 2-х условий **перечислимого объекта**, например, реализовать `IEnumerable<T>`:

```csharp
class FibonachiCollection : IEnumerable<int> {
    private readonly int _count;

    public FibonachiCollection(int count) {
        _count = count;
    }

    public IEnumerator<int> GetEnumerator() {
        return Fibonachi(_count).GetEnumerator();
    }

    IEnumerator IEnumerable.GetEnumerator() {
        return GetEnumerator();
    }

    IEnumerable<int> Fibonachi(int count) {
        for (int i = 0, prevFib = 1, curFib = 1; i < count; i++) {
            yield return prevFib;
            int newFib = prevFib + curFib;
            prevFib = curFib;
            curFib = newFib;
        }
    }
}
```

Мы объявили класс `FibonachiCollection`, конструктор которого принимает максимальное кол-во элементов, которые мы хотим получить в процессе итерации по нему (это ограничение выдумано нами и реализовано непосредственно в условии цикла `for`).

Этого класса достаточно, чтобы уже начать его использовать следующим образом:

```csharp
foreach (var i in new FibonachiCollection(20)) {
    Console.WriteLine(i);
}
```

Сама же эта конструкция будет преобразована в следующий вид:

```csharp
var enumerator = new FibonachiCollection(20).GetEnumerator();

/* Depends on exists of IDisposable implementation */
using (enumerator) {
    while (enumerator.MoveNext()) {
        Console.WriteLine(enumerator.Current);
    }
}
```

В случае если наш **перечислимый объект** реализовывает `IDisposable`, то код итерации так же оборачивается в конструкцию `using`.

## Чётность как extension метод

Для решения задачи по ограничению итоговой коллекции исключительно чётными числами, я предпочитаю использовать extension метод (отличный пример кода, который может быть переиспользован в дальнейшем):

```csharp
static class EnumerableExtensions {
    public static IEnumerable<int> Odd(this IEnumerable<int> numerics) {
        foreach (var num in numerics) {
            if (num%2 == 0)
                yield return num;
        }
    }
}
```

Здесь всё просто: статический метод расширяющий тип `this IEnumerable<int> numerics` методом `Odd()`.

Как итог:

```csharp
foreach (var i in new FibonachiCollection(20).Odd()) {
    Console.WriteLine(i);
}
```

## P.S.

В качестве факультатива всем читателям предлагаю реализовать (и поделиться в комментариях для сравнения) своей реализацией **итерируемого объекта**, вычисляющего значения асинхронно:

```csharp
foreach (var i in new AsyncFibonachiCollection(20).OddAsync()) {
    Console.WriteLine(await i);
}
```

**SPOILER:**
У меня получилось вот так :)

Коллекция:

```csharp
class AsyncFibonachiCollection : IEnumerable<Task<int>> {
    private readonly int _count;

    public AsyncFibonachiCollection(int count) {
        _count = count;
    }

    public IEnumerator<Task<int>> GetEnumerator() {
        return AsyncFibonachi(_count).GetEnumerator();
    }

    IEnumerator IEnumerable.GetEnumerator() {
        return GetEnumerator();
    }

    IEnumerable<Task<int>> AsyncFibonachi(int count) {
        for (int i = 0, prevFib = 1, curFib = 1; i < count; i++) {
            yield return Task.FromResult(prevFib);
            int newFib = prevFib + curFib;
            prevFib = curFib;
            curFib = newFib;
        }
    }
}
```

Методы:

```csharp
public static IEnumerable<Task<int>> OddAsync(this IEnumerable<Task<int>> numerics) {
    foreach (var num in numerics) {
        var result = num.Result;
        var isOdd = result%2 == 0;
        if (isOdd) {
            yield return Task.FromResult(result);
        }
    }
}
```

Ссылки по теме:

1. [C# 6.0 in a Nutshell](http://www.albahari.com/nutshell/);
