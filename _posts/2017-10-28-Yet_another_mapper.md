---
title: Пишем свой маппер для .NET Standard 2.0
tags: .NET C# performance
redirect_from: "/Yet_another_mapper/"
---

В сегодняшней заметке я хотел бы поведать вам о коротком приключении по написанию своего маппера для .NET Standard 2.0. Ссылка на github и результаты benchmark'ов прилагаются.

Думаю ни для кого из вас не секрет, что такое mapper и для чего он нужен. Буквально на каждом шагу в процессе работы мы сталкиваемся с теми или иными примерами маппингов (или трансформаций) данных из одного вида в другой. К ним можно отнести маппинг записей из хранилища в domain model, маппинг response удалённого сервиса в view model и уже затем в domain model и т.д. Зачастую, на границе уровня абстракции существуют входной и выходной форматы данных и именно в моменты взаимодействия абстракций такая вещь, как маппер, может показать себя во всей красе, привнося с собой существенную экономию времени и effort'ов для разработчика и, как следствие, забирая на себя долю от общей производительности системы.

Исходя из этого можно описать и MVP требования:

1. Скорость работы (less performance & memory impact);
2. Простота использования (clean & easy to use API).

Что касается первого пункта, то в этом нам поможет BenchmarkDotNet и вдумчивая реализация, не лишённая и оптимизаций. Для второго же я написал простой unit test, который, в некотором роде, выступает документацией API нашего маппера:

```csharp
[TestMethod]
public void WhenMappingExist_Then_Map()
{
    var dto = new CustomerDto
    {
        Id = 42,
        Title = "Test",
        CreatedAtUtc = new DateTime(2017, 9, 3),
        IsDeleted = true
    };

    mapper.Register<CustomerDto, Customer>();

    var customer = mapper.Map<CustomerDto, Customer>(dto);

    Assert.AreEqual(42, customer.Id);
    Assert.AreEqual("Test", customer.Title);
    Assert.AreEqual(new DateTime(2017, 9, 3), customer.CreatedAtUtc);
    Assert.AreEqual(true, customer.IsDeleted);
}
```

Итого, нам потребуется реализовать лишь 2 простых метода:

1. `void Register<TSource, TDest>()`;
2. `TDest Map<TSource, TDest>(TSource source)`.

## Регистрация

На самом деле процесс регистрации может осуществляться и при первом вызове метода `Map`, тем самым став лишним. Однако, я вынес его отдельно по следующим причинам:

1. Для верификации- в случае отсутствия конструктора по умолчанию (или невозможности осуществить маппинг итогового типа) на мой взгляд, сообщить об этом следует как можно раньше на этапе конфигурации, соблюдая тем самым принцип Fail fast. В противном случае ошибка невозможности создания экземпляра типа может настигнуть нас уже на этапе выполнения инфраструктурного кода или бизнес-логики;
2. Для расширения- на данный момент API предельно прост и under the hood подразумевает маппинг опирающийся на naming conventions, однако, вполне вероятно что уже очень скоро мы захотим вводить правила осуществления маппинга тех или иных полей, значением для присваивания которых может и вовсе явиться результат выполнения метода. В этом случае, дабы так же соблюсти и принцип Single responsible, такое разделение мне кажется вполне закономерным.

Если метод `Map` в любом маппере является основным и именно на него приходится львиная доля времени выполнения, то метод `Register` наоборот, для каждой пары типов вызывается лишь единожды на этапе конфигурации. Именно поэтому он является отличным кандидатом для совершения всех необходимых "тяжеловесных" манипуляций: генерации оптимального плана выполнения маппинга и как следствие, дальнейшего кеширования полученных результатов.

Таким образом, его реализация должна включать:

1. Построение плана выполнения создания и инициализации экземпляра требуемого типа;
2. Кеширование результатов.

## План выполнения

В C# нам доступно не так много способов создать и проинициализировать экземпляр типа в runtime и чем выше уровень абстракции того или иного метода, тем менее оптимальным с точки зрения времени выполнения он является. Ранее я уже сталкивался с подобным выбором в другом своём небольшом проекте под названием [FsContainer](https://github.com/FSou1/FsContainer) и потому следующие результаты не стали для меня удивительными.

```

BenchmarkDotNet=v0.10.9, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i5-5200U CPU 2.20GHz (Broadwell), ProcessorCount=4
Frequency=2143473 Hz, Resolution=466.5326 ns, Timer=TSC
.NET Core SDK=2.0.0
  [Host]     : .NET Core 2.0.0 (Framework 4.6.00001.0), 64bit RyuJIT
  DefaultJob : .NET Core 2.0.0 (Framework 4.6.00001.0), 64bit RyuJIT
```

```
 |                      Method |       Mean |     Error |    StdDev |     Median |
 |---------------------------- |-----------:|----------:|----------:|-----------:|
 | ExpressionCtorObjectBuilder |   8.548 ns | 0.2764 ns | 0.4541 ns |   8.608 ns |
 |     ActivatorCreateInstance |  79.379 ns | 1.6812 ns | 3.1987 ns |  78.890 ns |
 |       ConstructorInfoInvoke | 164.445 ns | 3.3355 ns | 4.3371 ns | 164.016 ns |
 |    DynamicMethodILGenerator |   5.859 ns | 0.2455 ns | 0.3015 ns |   5.819 ns |
 |                     NewCtor |   6.989 ns | 0.2615 ns | 0.5741 ns |   6.756 ns |
```

Несмотря на то, что использовать `ConstructorInfo.Invoke` и `Activator.CreateInstance` довольно легко, в данном списке с большим отрывом они являются явными аутсайдерами ввиду того, что в деталях своих реализаций они используют `RuntimeType` и `System.Reflection`. Это вполне приемлимо в повседневных задачах, но совершенно неуместно в рамках наших требований, где создание экземпляра типа является наиболее узким bottle neck'ом с точки зрения performance.

Что касается использования `Expression` и `DynamicMethod`, то здесь без сюрпризов- результатом выполнения являются указатели на скомпилированные функции, которые останется лишь вызвать, передав соответствующие аргументы.

Хотя Delegate, скомпилированный посредством генерации IL code налету, отрабатывает несколько быстрее, он не включает в себя код инициализации экземпляра типа. Более того, лично для меня воспроизведение IL инструкций посредством `ilgen.Emit` является весьма нетривиальным занятием.

```csharp
var dynamicMethod = new DynamicMethod("Create_" + ctorInfo.Name, ctorInfo.DeclaringType, new[] { typeof(object[]) });
var ilgen = dynamicMethod.GetILGenerator();
ilgen.Emit(OpCodes.Newobj, ctorInfo);
ilgen.Emit(OpCodes.Ret);
return dynamicMethod.CreateDelegate(typeof(Func<TDest>));
```

Именно поэтому я остановился на реализации с использованием `Expression`:

```csharp
var body = Expression.MemberInit(
    Expression.New(typeof(TDest)), props
);

return Expression.Lambda<Func<TSource, TDest>>(body, orig).Compile();
```

## Кеширование

Для кеширования скомпилированного делегата, который в дальнейшем будет использоваться для выполнения маппинга, я выбирал между `Dictionary` и `Hashtable`. Забегая вперёд, хотелось бы отметить, что ключевые роли играют не только тип коллекции, но и тип ключа, по которому будет осуществляться выборка. Для проверки этого утверждения был написан отдельный benchmark и получены следующие результаты:

```

BenchmarkDotNet=v0.10.9, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i5-5200U CPU 2.20GHz (Broadwell), ProcessorCount=4
Frequency=2143473 Hz, Resolution=466.5326 ns, Timer=TSC
.NET Core SDK=2.0.0
  [Host]     : .NET Core 2.0.0 (Framework 4.6.00001.0), 64bit RyuJIT
  DefaultJob : .NET Core 2.0.0 (Framework 4.6.00001.0), 64bit RyuJIT
```

```
 |              Method |      Mean |     Error |    StdDev |
 |-------------------- |----------:|----------:|----------:|
 |     DictionaryTuple |  80.37 ns | 1.6473 ns | 1.6179 ns |
 | DictionaryTypeTuple |  49.35 ns | 0.6235 ns | 0.5832 ns |
 |      HashtableTuple | 103.07 ns | 2.6081 ns | 2.4397 ns |
 |  HashtableTypeTuple |  71.51 ns | 0.8679 ns | 0.7694 ns |
```

Принимая это во внимание, можно сделать следующие заключения:

1. Использование типа `Dictionary` предпочтительнее `Hashtable` с точки зрения временных затрат на получение элемента коллекции;
2. Использование в качестве ключа типа `TypeTuple` ([src](https://github.com/MapsterMapper/Mapster/blob/57712b0fe6f381d9d3fd0e5c1dc539047e6fadc7/src/Mapster/Models/TypeTuple.cs)) предпочтительнее `Tuple<Type, Type>` с точки зрения временных затрат на `Equals` & `GetHashCode`;

## Маппинг

Внутренняя реализация метода `Map` должна быть предельно проста и оптимизирована ввиду того, что именно этот метод будет вызываться в 99.9% случаев. Поэтому всё, что нам необходимо сделать, это максимально быстро найти ссылку на скомпилированный ранее `Delegate` в кеше и вернуть результат его выполнения:

```csharp
public TDest Map<TSource, TDest>(TSource source)
{
    var key = new TypeTuple(typeof(TSource), typeof(TDest));
    var activator = GetMap(key);
    return ((Func<TSource, TDest>)activator)(source);
}
```

## Результаты

В качестве результатов хотелось бы привести итоги финальных замеров существующих (и находящихся в актуальном состоянии) на данный момент мапперов:

```

BenchmarkDotNet=v0.10.9, OS=Windows 8.1 (6.3.9600)
Processor=Intel Core i5-5200U CPU 2.20GHz (Broadwell), ProcessorCount=4
Frequency=2143473 Hz, Resolution=466.5326 ns, Timer=TSC
.NET Core SDK=2.0.0
  [Host]     : .NET Core 2.0.0 (Framework 4.6.00001.0), 64bit RyuJIT
  DefaultJob : .NET Core 2.0.0 (Framework 4.6.00001.0), 64bit RyuJIT
```

```
 |                 Method |       Mean |     Error |    StdDev |
 |----------------------- |-----------:|----------:|----------:|
 |      FsMapperBenchmark |  84.492 ns | 1.6972 ns | 1.6669 ns |
 | ExpressMapperBenchmark | 251.161 ns | 4.6736 ns | 4.3717 ns |
 |    AutoMapperBenchmark | 204.142 ns | 4.2002 ns | 9.1309 ns |
 |       MapsterBenchmark |  90.949 ns | 1.6393 ns | 1.4532 ns |
 |   AgileMapperBenchmark | 218.021 ns | 3.0921 ns | 2.7410 ns |
 |    CtorMapperBenchmark |   7.806 ns | 0.2472 ns | 0.2312 ns |
```

Исходный код проекта доступен на github: [https://github.com/FSou1/FsMapper](https://github.com/FSou1/FsMapper).

Спасибо что дочитали до конца и, надеюсь, эта заметка была вам полезна. Пишите в комментариях, что на ваш взгляд ещё можно было бы оптимизировать.
