---
title: История оптимизации одного IoC контейнера
tags: .NET C# IoC performance
redirect_from: "/IoC_container_optimization_story/"
---

В этой заметке мне хотелось бы поделиться информацией о небольшом, но, на мой взгляд, весьма и весьма полезном [проекте](https://github.com/stebet/DependencyInjectorBenchmarks), в котором [Stefán Jökull Sigurðarson](https://github.com/stebet) добавляет все известные ему IoC контейнеры, которые мигрировали на .NET Core, и с использованием [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet) проводит замеры instance resolving performance. Не упустил возможности поучавствовать в этом соревновании и я со своим маленьким проектом [FsContainer](https://github.com/FSou1/FsContainer).

## 1.2.0

После миграции проекта на .NET Core (хочу заметить, что это оказалось совершенно не сложно) сказать что я не пал духом, значит ничего не сказать и связано это было с тем, что один из трех замеров мой контейнер не проходил. В прямом значении этого слова- замер просто-напросто длился свыше 20 минут и не завершался.

Причина оказалась в этом участке кода:

```csharp
public object Resolve(Type type)
{
    var instance = _bindingResolver.Resolve(this, GetBindings(), type);

    if (!_disposeManager.Contains(instance))
    {
        _disposeManager.Add(instance);
    }

    return instance;
}
```

Если задуматься, основной принцип работы benchmark'ов- измерение количества выполняемых операций за еденицу времени (опционально потребляемую память), а значит, метод `Resolve` запускается максимально возможное количество раз. Вы можете заметить, что после resolve полученный instance добавляется в `_disposeManager` для дальнейшего его уничтожения в случае `container.Dispose()`. Т.к. внутри реализации находится `List<object>`, экземпляры в который добавляются посредством проверки на `Contains`, то можно догадаться, что налицо сразу 2 side-effect'a:

1. Каждый новый созданный экземпляр, используя проверку `Contains`, будет вычислять `GetHashCode` и искать среди ранее добавленных дубликат;
2. Т.к. каждый новый созданный экземпляр всегда будет являться уникальным (тестировался resolve с `TransientLifetimeManager`), то и размер `List<object>` будет постоянно увеличиваться посредством выделения нового, в 2 раза большего участка памяти и [копирования в него ранее добавленных элементов](http://referencesource.microsoft.com/#mscorlib/system/collections/generic/list.cs,115) (для добавления миллиона экземпляров операции выделения памяти и копирования будут вызваны минимум 20 раз);

Признаться, я не уверен какое решение является наиболее корректным в данном случае, ведь в реальной жизни мне сложно представить, когда один контейнер будет держать у себя миллионы ссылок на ранее созданные экземпляры, поэтому я решил лишь половину проблемы, добавив (вполне логичное) ограничение на добавление в `_disposeManager` лишь тех объектов, которые реализуют `IDisposable`.

```csharp
if (instance is IDisposable && !_disposeManager.Contains(instance))
{
    _disposeManager.Add(instance);
}
```

Как итог, замер завершился за вполне приемлимое время и выдал следующие результаты:

```
 |         Method |         Mean |       Error |      StdDev | Scaled | ScaledSD |  Gen 0 |  Gen 1 | Allocated |
 |--------------- |-------------:|------------:|------------:|-------:|---------:|-------:|-------:|----------:|
 |         Direct |     13.77 ns |   0.3559 ns |   0.3655 ns |   1.00 |     0.00 | 0.0178 |      - |      56 B |
 |    LightInject |     36.95 ns |   0.1081 ns |   0.0902 ns |   2.69 |     0.07 | 0.0178 |      - |      56 B |
 | SimpleInjector |     46.17 ns |   0.2746 ns |   0.2434 ns |   3.35 |     0.09 | 0.0178 |      - |      56 B |
 |     AspNetCore |     71.09 ns |   0.4592 ns |   0.4296 ns |   5.17 |     0.14 | 0.0178 |      - |      56 B |
 |        Autofac |  1,600.67 ns |  14.4742 ns |  12.8310 ns | 116.32 |     3.10 | 0.5741 |      - |    1803 B |
 |   StructureMap |  1,815.87 ns |  18.2271 ns |  16.1578 ns | 131.95 |     3.55 | 0.6294 |      - |    1978 B |
 |    FsContainer |  2,819.01 ns |   6.0161 ns |   5.3331 ns | 204.85 |     5.24 | 0.4845 |      - |    1524 B |
 |        Ninject | 12,812.70 ns | 255.5191 ns | 447.5211 ns | 931.06 |    39.95 | 1.7853 | 0.4425 |    5767 B |
```

Доволен ими я конечно же не стал и приступил к поиску дальнейших способов оптимизации.

## 1.2.1

В текущей версии контейнера определение необходимого конструктора и требуемых для него аргументов является неизменным, следовательно, эту информацию можно закешировать и впредь не тратить процессорное время. Результатом этой оптимизации стало добавление `ConcurrentDictionary`, ключём которого является запрашиваемый тип (`Resolve<T>`), а значениями- конструктор и аргументы, которые будут использоваться для создания экземпляра непосредственно.

```csharp
private readonly IDictionary<Type, Tuple<ConstructorInfo, ParameterInfo[]>> _ctorCache =
    new ConcurrentDictionary<Type, Tuple<ConstructorInfo, ParameterInfo[]>>();
```

Судя по проведённым замерам, такая нехитрая операция увеличила производительность более чем на 30%:

```
 |         Method |         Mean |       Error |      StdDev | Scaled | ScaledSD |  Gen 0 |  Gen 1 |  Gen 2 | Allocated |
 |--------------- |-------------:|------------:|------------:|-------:|---------:|-------:|-------:|-------:|----------:|
 |         Direct |     13.50 ns |   0.2240 ns |   0.1986 ns |   1.00 |     0.00 | 0.0178 |      - |      - |      56 B |
 |    LightInject |     36.94 ns |   0.0999 ns |   0.0886 ns |   2.74 |     0.04 | 0.0178 |      - |      - |      56 B |
 | SimpleInjector |     46.40 ns |   0.3409 ns |   0.3189 ns |   3.44 |     0.05 | 0.0178 |      - |      - |      56 B |
 |     AspNetCore |     70.26 ns |   0.4897 ns |   0.4581 ns |   5.21 |     0.08 | 0.0178 |      - |      - |      56 B |
 |        Autofac |  1,634.89 ns |  15.3160 ns |  14.3266 ns | 121.14 |     2.01 | 0.5741 |      - |      - |    1803 B |
 |    FsContainer |  1,779.12 ns |  18.9507 ns |  17.7265 ns | 131.83 |     2.27 | 0.2441 |      - |      - |     774 B |
 |   StructureMap |  1,830.01 ns |   5.4174 ns |   4.8024 ns | 135.60 |     1.97 | 0.6294 |      - |      - |    1978 B |
 |        Ninject | 12,558.59 ns | 268.1920 ns | 490.4042 ns | 930.58 |    38.29 | 1.7858 | 0.4423 | 0.0005 |    5662 B |
```

## 1.2.2

Проводя замеры, BenchmarkDotNet уведомляет пользователя о том, что та или иная сборка может быть не оптимизирована (собрана в конфигурации Debug). Я долго не мог понять, почему это сообщение высвечивалось в проекте, где контейнер подключался посредством nuget package и, какого же было моё удивление, когда я увидел возможный список параметров для nuget pack:

```
nuget pack MyProject.csproj -properties Configuration=Release
```

Оказывается, всё это время я собирал package в конфигурации Debug, что судя по обновленным результатам замеров замедляло производительность ещё аж на 25%.

```
 |         Method |         Mean |       Error |      StdDev | Scaled | ScaledSD |  Gen 0 |  Gen 1 |  Gen 2 | Allocated |
 |--------------- |-------------:|------------:|------------:|-------:|---------:|-------:|-------:|-------:|----------:|
 |         Direct |     13.38 ns |   0.2216 ns |   0.2073 ns |   1.00 |     0.00 | 0.0178 |      - |      - |      56 B |
 |    LightInject |     36.85 ns |   0.0577 ns |   0.0511 ns |   2.75 |     0.04 | 0.0178 |      - |      - |      56 B |
 | SimpleInjector |     46.56 ns |   0.5329 ns |   0.4724 ns |   3.48 |     0.06 | 0.0178 |      - |      - |      56 B |
 |     AspNetCore |     70.17 ns |   0.1403 ns |   0.1312 ns |   5.25 |     0.08 | 0.0178 |      - |      - |      56 B |
 |    FsContainer |  1,271.81 ns |   4.0828 ns |   3.8190 ns |  95.09 |     1.44 | 0.2460 |      - |      - |     774 B |
 |        Autofac |  1,648.52 ns |   2.3197 ns |   2.0563 ns | 123.26 |     1.84 | 0.5741 |      - |      - |    1803 B |
 |   StructureMap |  1,829.05 ns |  17.8238 ns |  16.6724 ns | 136.75 |     2.37 | 0.6294 |      - |      - |    1978 B |
 |        Ninject | 12,520.08 ns | 248.2530 ns | 534.3907 ns | 936.10 |    41.98 | 1.7860 | 0.4423 | 0.0008 |    5662 B |
```

## 1.2.3

Ещё одной оптимизацией стало кеширование функции активатора, которая компилируется с использованием Expression:

```csharp
private readonly IDictionary<Type, Func<object[], object>> _activatorCache =
    new ConcurrentDictionary<Type, Func<object[], object>>();
```

Универсальная функция принимает в качестве аргументов `ConstructorInfo` и массив аргументов `ParameterInfo[]`, а в качестве результата возвращает строго типизированную lambda:

```csharp
private Func<object[], object> GetActivator(ConstructorInfo ctor, ParameterInfo[] parameters) {
    var p = Expression.Parameter(typeof(object[]), "args");
    var args = new Expression[parameters.Length];

    for (var i = 0; i < parameters.Length; i++)
    {
        var a = Expression.ArrayAccess(p, Expression.Constant(i));
        args[i] = Expression.Convert(a, parameters[i].ParameterType);
    }

    var b = Expression.New(ctor, args);
    var l = Expression.Lambda<Func<object[], object>>(b, p);

    return l.Compile();
}
```

Соглашусь, что логичным продолжением этого решения должно стать компилирование всей функции Resolve, а не только Activator, но даже в текущей реализации это привнесло 10% ускорение, тем самым позволив занять уверенное 5-е место:

```
 |         Method |         Mean |       Error |      StdDev | Scaled | ScaledSD |  Gen 0 |  Gen 1 |  Gen 2 | Allocated |
 |--------------- |-------------:|------------:|------------:|-------:|---------:|-------:|-------:|-------:|----------:|
 |         Direct |     13.24 ns |   0.0836 ns |   0.0698 ns |   1.00 |     0.00 | 0.0178 |      - |      - |      56 B |
 |    LightInject |     37.39 ns |   0.0570 ns |   0.0533 ns |   2.82 |     0.01 | 0.0178 |      - |      - |      56 B |
 | SimpleInjector |     46.22 ns |   0.2327 ns |   0.2063 ns |   3.49 |     0.02 | 0.0178 |      - |      - |      56 B |
 |     AspNetCore |     70.53 ns |   0.2885 ns |   0.2698 ns |   5.33 |     0.03 | 0.0178 |      - |      - |      56 B |
 |    FsContainer |  1,038.13 ns |  17.1037 ns |  15.9988 ns |  78.41 |     1.23 | 0.2327 |      - |      - |     734 B |
 |        Autofac |  1,551.33 ns |   3.6293 ns |   3.2173 ns | 117.17 |     0.64 | 0.5741 |      - |      - |    1803 B |
 |   StructureMap |  1,944.35 ns |   1.8665 ns |   1.7459 ns | 146.85 |     0.76 | 0.6294 |      - |      - |    1978 B |
 |        Ninject | 13,139.70 ns | 260.8754 ns | 508.8174 ns | 992.43 |    38.35 | 1.7857 | 0.4425 | 0.0004 |    5682 B |
```

# 1.2.4

Уже после публикации статьи [@turbanoff](https://github.com/turbanoff) заметил, что в случае с `ConcurrentDictionary` производительность метода `GetOrAdd` выше, чем у ContainsKey/Add, за что ему отдельное спасибо. Результаты замеров представлены ниже:

До:

```csharp
if (!_activatorCache.ContainsKey(concrete)) {
    _activatorCache[concrete] = GetActivator(ctor, parameters);
}
```

```
 |           Method |       Mean |      Error |    StdDev |     Median |  Gen 0 | Allocated |
 |----------------- |-----------:|-----------:|----------:|-----------:|-------:|----------:|
 | ResolveSingleton |   299.0 ns |   7.239 ns |  19.45 ns |   295.7 ns | 0.1268 |     199 B |
 | ResolveTransient |   686.3 ns |  32.333 ns |  86.30 ns |   668.7 ns | 0.2079 |     327 B |
 |  ResolveCombined | 1,487.4 ns | 101.057 ns | 273.21 ns | 1,388.7 ns | 0.4673 |     734 B |
```

После:

```csharp
var activator = _activatorCache.GetOrAdd(concrete, x => GetActivator(ctor, parameters));
```

```
 |           Method |       Mean |     Error |    StdDev |  Gen 0 | Allocated |
 |----------------- |-----------:|----------:|----------:|-------:|----------:|
 | ResolveSingleton |   266.6 ns |  4.955 ns |  4.393 ns | 0.1268 |     199 B |
 | ResolveTransient |   512.0 ns | 16.974 ns | 16.671 ns | 0.3252 |     511 B |
 |  ResolveCombined | 1,119.2 ns | 18.218 ns | 15.213 ns | 0.6943 |    1101 B |
```

## P.S.

В качестве эксперимента я решил произвести замеры времени создания объектов используя разные конструкции. Сам проект доступен на [Github](https://github.com/FSou1/NetBenchmarking), а результаты вы можете видеть ниже. Для полноты картины не хватает только способа активации посредством генерации IL инструкций максимально приближенных к методу Direct- именно этот способ используют контейнеры из топ 4, что и позволяет им добиваться таких впечатляющих результатов.

```
 |                  Method |       Mean |     Error |    StdDev |  Gen 0 | Allocated |
 |------------------------ |-----------:|----------:|----------:|-------:|----------:|
 |                  Direct |   4.031 ns | 0.1588 ns | 0.1890 ns | 0.0076 |      24 B |
 |          CompiledInvoke |  85.541 ns | 0.5319 ns | 0.4715 ns | 0.0178 |      56 B |
 |   ConstructorInfoInvoke | 316.088 ns | 1.8337 ns | 1.6256 ns | 0.0277 |      88 B |
 | ActivatorCreateInstance | 727.547 ns | 2.9228 ns | 2.5910 ns | 0.1316 |     416 B |
 |           DynamicInvoke | 974.699 ns | 5.5867 ns | 5.2258 ns | 0.0515 |     168 B |
```
