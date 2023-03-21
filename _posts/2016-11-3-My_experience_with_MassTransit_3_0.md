---
title: Опыт использования MassTransit 3.0
tags: .NET C# MassTransit RabbitMQ
redirect_from: "/My_experience_with_MassTransit_3_0/"
---

MassTransit это open source библиотека, разработанная на языке C# для .NET платформы, упрощающая работу с шиной данных, которая используется при построении распределенных приложений и реализации SOA (service oriented architecture).

В качестве message broker могут выступать RabbitMq, Azure Service Bus или In-Memory менеджер (в случае с In-Memory область видимости ограничивается процессом, в котором проинициализирован экземпляр).

**Содержание**:

<ul>
<li><a href="#command-event">Команды и события</a>
<ul>
<li><a href="#command">Команды</a></li>
<li><a href="#event">События</a></li></ul></li>
<li><a href="#contract">Контракты сообщений</a></li>
<li><a href="#routing">Роутинг</a>
<ul>
<li><a href="#exchange">Exchange</a></li>
<li><a href="#message-body">Формат сообщения</a></li></ul></li>
<li><a href="#consumer">Консьюмеры (Consumer)</a></li>
<li><a href="#di">Конфигурация контейнера DI</a></li>
<li><a href="#observer">Наблюдатели (Observer)</a></li>
<li><a href="#new-30">Новое в MassTransit 3.0</a></li>
<li><a href="#ps">Заключение</a></li>
</ul>

<a name="command-event"></a>

## Команды и события

В библиотеке заложено 2 основных типа сообщений: команды и события.

<a name="command"></a>

### Команды

Сигнализируют о необходимости выполнить некое действие. Для наиболее содержательного наименования команды желательно использовать структуру глагол + существительное:
EstimateConnection, SendSms, NotifyCustomerOrderProcessed.

Работа с командами осуществляется с помощью метода Send (интерфейса ISendEndpoint) и указания получателя endpoint (очереди):

```csharp
private static async Task SendSmsCommand(IBus busControl)
{
   var command = new SendSmsCommand
   {
       CommandId = Guid.NewGuid(),
       PhoneNumber = "89031112233",
       Message = "Thank you for your reply"
   };

   var endpoint = await busControl.GetSendEndpoint(AppSettings.CommandEndpoint);
   await endpoint.Send(command);
}
```

<a name="event"></a>

### События

Сигнализируют о случившемся событии, которое может быть интересно некоему набору подписчиков (паттерн Observer), которые на эти события реагируют, например: ConnectionEstimated, CallTerminated, SmsSent, CustomerNotified.

Работа с событиями осуществляется с помощью метода Publish (интерфейса [IPublishEndpoint](https://github.com/MassTransit/MassTransit/blob/develop/src/MassTransit/ISendEndpoint.cs)).

В терминологии заложено и основное различие этих типов сообщений — команда доставляется единственному исполнителю (дабы избежать дублирования выполнения):

![mt command](/assets/images/posts/mt_command.png)

Изображение из статьи [MassTransit Send vs. Publish](https://www.maldworth.com/2015/10/27/masstransit-send-vs-publish/)

В то время как событие ориентировано на оповещение n-подписчиков, каждый из которых реагирует на случившееся событие по-своему.

![mt event](/assets/images/posts/mt_event.png)

Изображение из статьи [MassTransit Send vs. Publish](https://www.maldworth.com/2015/10/27/masstransit-send-vs-publish/)

Иными словами, при запущенных n-консьюмеров (от англ. consumer — потребитель, обработчик), обрабатывающих команду, после её публикации только один из них получит сообщение о ней, в то время как сообщение о событии получит каждый.

<a name="contract"></a>

## Контракты сообщений

Согласно документации MassTransit, при объявлении контрактов сообщений [рекомендуется](http://docs.masstransit-project.com/en/latest/usage/messages.html) прибегать к интерфейсам:

```csharp
public interface ISendSms {
  Guid CommandId { get; }
  string PhoneNumber { get; }
  string Message { get; }
}
```

Как упоминалось ранее, отправка команд должна осуществляться исключительно с помощью метода Send (интерфейса IBus) и указания адресата (endpoint).

```csharp
public interface ISmsSent {
  Guid EventId { get; }
  DateTime SentAtUtc { get; }
}
```

События отправляются с помощью метода Publish.

<a name="routing"></a>

## Роутинг

Как распределение сообщений по exchange, так и выбор консьюмеров (о них в этой статье будет рассказано чуть позже) для обработки базируются на runtime типах этих сообщений,- в наименовании используются namespace и имя типа, в случае с generic имя родительского типа и перечень аргументов.

<a name="exchange"></a>

### Exchange

При [конфигурации receive endpoint‘a](https://github.com/MassTransit/MassTransit/blob/699b167645b7c6f2df10483378420f586b358dc4/src/MassTransit/Configuration/ConsumeConfigurators/HandlerConfigurator.cs#L53) (подключении ранее зарегистрированных консьюмеров) в случае использования в качестве канала доставки сообщений RabbitMq на основании указанных к обработке консьюмерами типов сообщений [формируются](https://github.com/MassTransit/MassTransit/blob/b488f08beed692e0a9acaa6a45fb022a7ce457bc/src/MassTransit.RabbitMqTransport/Topology/RabbitMqExchangeBindingExtensions.cs#L24) наименования требуемых exchange, в которые затем эти самые сообщения и будут помещаться.

Аналогичные действия на этапе [конфигурации send endpoint‘a](https://github.com/MassTransit/MassTransit/blob/f3dbf6d0377d6494cffd02fa10e0eb50d0337fb9/src/MassTransit.RabbitMqTransport/Transport/RabbitMqPublishEndpointProvider.cs#L96) выполняются и для команд, для отправки которых также требуются собственные exchange.

На изображении можно увидеть созданные в рамках моего сценария exchange:

![mt exchange](/assets/images/posts/mt_exchange.png)

В том случае, если конфигурируя receive endpoint мы указываем наименование очереди:

```csharp
cfg.ReceiveEndpoint(host, "play_with_masstransit_queue", e => e.LoadFrom(container));
```

то в привязках exchange сообщений можно будет увидеть следующую картину:

![mt_config_queue](/assets/images/posts/mt_config_queue.png)

Итоговый путь сообщения, тип которого имплементирует ISmsEvent, будет иметь следующий вид:

![mt_queue_flow](/assets/images/posts/mt_queue_flow.png)

Если же конфигурация осуществляется без указания очереди:

```csharp
cfg.ReceiveEndpoint(host, e=> e.LoadFrom(container));
```

То имена для последнего exchange и очереди формируются автоматически, а по завершению работы они будут удалены:

![mt_without_queue_flow](/assets/images/posts/mt_without_queue_flow.png)

<a name="message-body"></a>

### Формат сообщения

Говоря о формате сообщения, хотелось бы подробнее остановиться на наименовании (или messageType). За его формирование (заголовков urn:message:) ответственна функция [GetUrnForType(Type type)](https://github.com/MassTransit/MassTransit/blob/b488f08beed692e0a9acaa6a45fb022a7ce457bc/src/MassTransit/MessageUrn.cs#L85). Для примера я добавил для команды ISendSms наследование от ICommand и generic тип:

```csharp
public interface ICommand<T>
{
}

public interface ISendSms<T> : ICommand<T>
{
   Guid CommandId { get; }
   string PhoneNumber { get; }
   string Message { get; }
}

class SendSmsCommand : ISendSms<string>
{
   public Guid CommandId { get; set; }
   public string PhoneNumber { get; set; }
   public string Message { get; set; }
}
```

Сформированное сообщение в таком случае будет содержать следующее значение в поле messageType (на основании которого после получения сообщения затем и выбирается ответственный за обработку консьюмер):

```javascript
"messageType": [
  "urn:message:PlayWithMassTransit30.Extension:SendSmsCommand",
  "urn:message:PlayWithMassTransit30.Contract.Command:ISendSms[[System:String]]",
  "urn:message:PlayWithMassTransit30.Contract.Command:ICommand[[System:String]]"
]
```

Кроме messageType сообщение содержит информацию о host, которым оно было отправлено:

```javascript
"host": {
  "machineName": "DESKTOP-SI9OHUR",
  "processName": "PlayWithMassTransit30.vshost",
  "processId": 1092,
  "assembly": "PlayWithMassTransit30",
  "assemblyVersion": "1.0.0.0",
  "frameworkVersion": "4.0.30319.42000",
  "massTransitVersion": "3.4.1.808",
  "operatingSystemVersion": "Microsoft Windows NT 6.2.9200.0"
}
```

Значимую часть payload:

```javascript
"message": {
  "commandId": "7388f663-82dc-403a-8bf9-8952f2ff262e",
  "phoneNumber": "89031112233",
  "message": "Thank you for your reply"
}
```

и иные служебные поля и заголовки.

<a name="consumer"></a>

## Консьюмеры (Consumer)

Консьюмер — это класс, который обрабатывает один или несколько типов сообщений, указываемых при объявлении в наследовании интерфейса IConsumer, где T это тип обрабатываемого данным консьюмером сообщения.

Пример консьюмера, обрабатывающего команду ISendSms и публикующего событие ISmsSent:

```csharp
public class SendSmsConsumer : IConsumer<ISendSms<string>>
{
   public SendSmsConsumer(IBusControl busControl)
   {
       _busControl = busControl;
   }

   public async Task Consume(ConsumeContext<ISendSms<string>> context)
   {
       var message = context.Message;

       Console.WriteLine($"[IConsumer<ISendSms>] Send sms command consumed");
       Console.WriteLine($"[IConsumer<ISendSms>] CommandId: {message.CommandId}");
       Console.WriteLine($"[IConsumer<ISendSms>] Phone number: {message.PhoneNumber}");
       Console.WriteLine($"[IConsumer<ISendSms>] Message: {message.Message}");

       Console.Write(Environment.NewLine);
       Console.WriteLine("Публикация события: Смс сообщение отправлено");
       await _busControl.SmsSent(DateTime.UtcNow);
   }

   private readonly IBus _busControl;
}
```

После того, как мы получили команду на отправку смс сообщения и выполнили требуемые действия, мы формируем и отправляем событие о том, что смс доставлено.

Код отправки сообщений я вынес в отдельный Extension класс над IBusControl, там же находится и имплементация самих сообщений:

```csharp
public static class BusExtensions
{
   /// <summary>
   /// Отправка смс сообщения
   /// </summary>
   /// <param name="bus"></param>
   /// <param name="host"></param>
   /// <param name="phoneNumber"></param>
   /// <param name="message"></param>
   /// <returns></returns>
   public static async Task SendSms(
       this IBus bus, Uri host, string phoneNumber, string message
   )
   {
       var command = new SendSmsCommand
       {
           CommandId = Guid.NewGuid(),
           PhoneNumber = phoneNumber,
           Message = message
       };

       await bus.SendCommand(host, command);
   }

   /// <summary>
   /// Публикация события об отправке смс сообщения
   /// </summary>
   /// <param name="bus"></param>
   /// <param name="sentAtUtc"></param>
   /// <returns></returns>
   public static async Task SmsSent(
       this IBus bus, DateTime sentAtUtc
   )
   {
       var @event = new SmsSentEvent
       {
           EventId = Guid.NewGuid(),
           SentAtUtc = sentAtUtc
       };

       await bus.PublishEvent(@event);
   }

   /// <summary>
   /// Отправка команды
   /// </summary>
   /// <typeparam name="T"></typeparam>
   /// <param name="bus"></param>
   /// <param name="address"></param>
   /// <param name="command"></param>
   /// <returns></returns>
   private static async Task SendCommand<T>(this IBus bus, Uri address, T command) where T : class
   {
       var endpoint = await bus.GetSendEndpoint(address);
       await endpoint.Send(command);
   }

   /// <summary>
   /// Публикация события
   /// </summary>
   /// <typeparam name="T"></typeparam>
   /// <param name="bus"></param>
   /// <param name="message"></param>
   /// <returns></returns>
   private static async Task PublishEvent<T>(this IBus bus, T message) where T : class
   {
       await bus.Publish(message);
   }
}

class SendSmsCommand : ISendSms<string>
{
   public Guid CommandId { get; set; }
   public string PhoneNumber { get; set; }
   public string Message { get; set; }
}

class SmsSentEvent : ISmsSent
{
   public Guid EventId { get; set; }
   public DateTime SentAtUtc { get; set; }
}
```

На мой взгляд, данное решение вполне удачно позволяет отделить код бизнес-логики от деталей реализации межсистемного (компонентного) взаимодействия и инкапсулировать их в одном месте.

<a name="di"></a>

## Конфигурация контейнера DI

На данный момент MassTransit предоставляет возможность использовать следующие [популярные контейнеры](http://docs.masstransit-project.com/en/latest/usage/containers/index.html):

1. Autofac;
2. Ninject;
3. StructureMap;
4. Unity;
5. Castle Windsor.

В случае с UnityContainer потребуется установить nuget package MassTransit.Unity, после чего станет доступен метод расширения LoadFrom:

```csharp
public static class UnityExtensions
{
    public static void LoadFrom(this IReceiveEndpointConfigurator configurator, IUnityContainer container);
}
```

Пример использования выглядит следующим образом:

```csharp
public static IBusControl GetConfiguredFactory(IUnityContainer container)
{
   if (container == null)
   {
       throw new ArgumentNullException(nameof(container));
   }

   var control = Bus.Factory.CreateUsingRabbitMq(cfg => {
       var host = cfg.Host(AppSettings.Host, h => { });


       // cfg.ReceiveEndpoint(host, e => e.LoadFrom(container));
       cfg.ReceiveEndpoint(host, "play_with_masstransit_queue", e => e.LoadFrom(container));
   });

   control.ConnectConsumeObserver(new ConsumeObserver());
   control.ConnectReceiveObserver(new ReceiveObserver());
   control.ConnectConsumeMessageObserver(new ConsumeObserverSendSmsCommand());
   control.ConnectSendObserver(new SendObserver());
   control.ConnectPublishObserver(new PublishObserver());

   return control;
}
```

В качестве срока жизни консьюмеров в контейнере [документация](http://docs.masstransit-project.com/en/latest/usage/containers/unity.html) предлагает использовать ContainerControlledLifetimeManager().

<a name="observer"></a>

## Наблюдатели (Observer)

Для мониторинга процесса обработки сообщений доступно подключение наблюдателей (Observer). Для этого MassTransit предоставляет следующий набор интерфейсов для обработчиков:

1. IReceiveObserver- срабатывает сразу же после получения сообщения и создания RecieveContext;
2. IConsumeObserver — срабатывает после создания ConsumeContext;
3. IConsumeMessageObserver<T> — для наблюдения за сообщениями типа T, в методах которого будет доступно строго-типизированное содержимое сообщения;
4. ISendObserver — для наблюдения за отправляемыми командами;
5. IPublishObserver — для наблюдения за отправляемыми событиями.

Для каждого из них интерфейс IBusControl предоставляет собственный метод подключения, выполнение которого должно быть осуществлено непосредственно перед IBusControl.Start().

В качестве примера далее представлена реализация ConsumeObserver:

```csharp
public class ConsumeObserver : IConsumeObserver
{
   public Task PreConsume<T>(ConsumeContext<T> context) where T : class
   {
       Console.WriteLine($"[ConsumeObserver] PreConsume {context.MessageId}");
       return Task.CompletedTask;
   }

   public Task PostConsume<T>(ConsumeContext<T> context) where T : class
   {
       Console.WriteLine($"[ConsumeObserver] PostConsume {context.MessageId}");
       return Task.CompletedTask;
   }

   public Task ConsumeFault<T>(ConsumeContext<T> context, Exception exception) where T : class
   {
       Console.WriteLine($"[ConsumeObserver] ConsumeFault {context.MessageId}");
       return Task.CompletedTask;
   }
}
```

Я не буду приводить код каждого из консьюмеров, т.к. по принципу работы и структуре они схожи. Имплементацию каждого из них можно посмотреть в [документации](http://docs.masstransit-project.com/en/latest/usage/observers.html) или в исходниках на [Github](https://github.com/FSou1/PlayWithMassTransit30/tree/master/PlayWithMassTransit30/Consumer).

Итоговый pipeline получения команды на отправку смс сообщения, её обработки и публикации события о её успешном выполнении выглядит следующим образом:

![mt_result](/assets/images/posts/mt_result.png)

<a name="new-30"></a>

## Новое в MassTransit 3.0

С изменениями, которые коснулись новой версии библиотеки, вы можете ознакомиться в 2-х обзорных статьях автора библиотеки Chris Patterson’а на страницах его блога: [MassTransit 3 API Changes](https://lostechies.com/chrispatterson/2015/02/24/masstransit-3-api-changes/) и [MassTransit v3 Update](https://lostechies.com/chrispatterson/2015/06/16/masstransit-v3-update/).

<a name="ps"></a>

## Заключение

Здесь должно было быть сравнение наиболее популярных библиотек для работы с очередями сообщений, однако, я решил оставить это для отдельной статьи.

Надеюсь, мне удалось провести для вас поверхностное знакомство с библиотекой MassTransit, за гранью которого ещё остаются такие интересные вещи, как транзакционность, персистентность (интеграция с NHibernate, MondoDb, EntityFramework), планировщик отправки сообщений (интеграция с Quartz), state machine (Automatonymous и Saga), логирование (Log4Net, NLog), многопоточность и многое другое.

Исходные коды примеров доступны на [Github](https://github.com/FSou1/PlayWithMassTransit30).

Используемые материалы:

1. [Документация MassTransit](http://docs.masstransit-project.com/en/latest/overview/backstory.html).
