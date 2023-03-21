---
title: Использование единого IoC Container'a в рамках HTTP запроса между Web API и OWIN Middleware
tags: ASP.NET IoC OWIN
redirect_from: "/Implement_unity_container_per_http_request_middleware/"
---

Целью данной статьи является поиск рабочего решение, которое позволяет иметь единый контейнер зависимостей (IoC контейнер) на протяжении всего жизненного цикла запроса, контролировать его создание и уничтожение. Это может понадобиться в том случае, если web-приложение должно иметь транзакционность (а на мой взгляд любое web-приложение его обязано иметь, т.е. применять изменения (например в БД) только в случае успешной обработки запроса и делать их отмену, если на любом из этапов возникла ошибка, свидетельствующая о некорректном результате и неконтролируемых последствиях) ([**GitHub source codes**](https://github.com/FSou1/UnityPerRequestMiddleware)).

## Теория

Проекты Web API 2 конфигурируются с помощью OWIN интерфейса IAppBuilder, который призван помочь построить pipeline обработки входящего запроса.

![owin pipeline](/assets/images/posts/owin-pipeline.png)

На изображении выше виден жизненный цикл запроса,- он проходит по всем компонентам цепочки, затем попадает в Web API (что также является отдельным компонентом) и возвращается обратно, формируя или декорируя ответ от сервера.

Для того, чтобы иметь единый контейнер зависимостей нам потребуется создавать его явно в начале обработки запроса и уничтожать по завершению:

1. Начало обработки запроса;
2. Создание контейнера;
3. Использование контейнера в Middleware;
4. Использование контейнера в Web API;
5. Уничтожение контейнера;
6. Завершение обработки запроса.

Для этого нам достаточно сконфигурировать контейнер, зарегистрировать его в Web API (посредством DependencyResolver):

```csharp
// Configure our parent container
var container = UnityConfig.GetConfiguredContainer();

// Pass our parent container to HttpConfiguration (Web API)
var config = new HttpConfiguration {
    DependencyResolver = new UnityDependencyResolver(container)
};

WebApiConfig.Register(config);
```

написать собственный Middleware, который будет создавать дочерний контейнер:

```csharp
public class UnityContainerPerRequestMiddleware : OwinMiddleware
{
    public UnityContainerPerRequestMiddleware(OwinMiddleware next, IUnityContainer container) : base(next)
    {
        _next = next;
        _container = container;
    }

    public override async Task Invoke(IOwinContext context)
    {
        // Create child container (whose parent is global container)
        var childContainer = _container.CreateChildContainer();

        // Set created container to owinContext (to become available at other places using OwinContext.Get<IUnityContainer>(key))
        context.Set(HttpApplicationKey.OwinPerRequestUnityContainerKey, childContainer);

        await _next.Invoke(context);

        // Dispose container that would dispose each of container's registered service
        childContainer.Dispose();
    }

    private readonly OwinMiddleware _next;
    private readonly IUnityContainer _container;
}
```

и использовать его в других Middleware’ах (в моей реализации я сохраняю контейнер в глобальном OwinContext с помощью context.Set, который передаётся в каждый следующий middleware и получаю его с помощью context.Get):

```csharp
public class CustomMiddleware : OwinMiddleware
{
    public CustomMiddleware(OwinMiddleware next) : base(next)
    {
        _next = next;
    }

    public override async Task Invoke(IOwinContext context)
    {
        // Get container that we set to OwinContext using common key
        var container = context.Get<IUnityContainer>(HttpApplicationKey.OwinPerRequestUnityContainerKey);

        // Resolve registered services
        var sameInARequest = container.Resolve<SameInARequest>();

        await _next.Invoke(context);
    }

    private readonly OwinMiddleware _next;
}
```

На этом можно было бы закончить, если бы не одно НО.

## Проблема

Middleware Web API внутри себя имеет свой собственный цикл обработки запроса, который выглядит следующим образом:

1. Запрос попадает в HttpServer для начала обработки HttpRequestMessage и передачи его в систему маршрутизации;
2. HttpRoutingDispatcher извлекает данные из запроса и с помощью таблицы Route’ов определяет контроллер, ответственный за обработку;
3. В HttpControllerDispatcher создаётся определённый ранее контроллер и вызывается метод обработки запроса с целью формирования HttpResponseMessage.

За создание контроллера отвечает следующая строка в DefaultHttpControllerActivator:

```csharp
IHttpController instance = (IHttpController)request.GetDependencyScope().GetService(controllerType);
```

Основное содержимое метода GetDependencyScope:

```csharp
public static IDependencyScope GetDependencyScope(this HttpRequestMessage request) {
    // …

    IDependencyResolver dependencyResolver = request.GetConfiguration().DependencyResolver;
    result = dependencyResolver.BeginScope();

    request.Properties[HttpPropertyKeys.DependencyScope] = result;
    request.RegisterForDispose(result);

    return result;
}
```

Из него видно, что Web API запрашивает DependencyResolver, который мы для него зарегистрировали в HttpConfiguration и с помощью dependencyResolver.BeginScope() создаёт дочерний контейнер, в рамках которого уже и будет создан экземпляр ответственного за обработку запроса контроллера.

Для нас это значит следующее: контейнер, который мы используем в наших Middleware’ах и в Web API не являются одними и теми же,- больше того, они находятся на одном уровне вложенности, где глобальный контейнер- их общий родитель, т.е.:

1. Глобальный контейнер;
2. Контейнер, созданный в UnityContainerPerRequestMiddleware;
3. Контейнер, созданный в Web API.

Для Web API это выглядит вполне логичным в том случае, когда оно является единственным местом обработки запроса,- контейнер создается вначале и уничтожается в конце (это ровно то, чего мы стараемся добиться).

Однако, в данный момент Web API является лишь одним из звеньев в pipeline, а значит от создания собственного контейнера придется отказаться,- нашей задачей является переопределить данное поведение и указать контейнер, в рамках которого Web API требуется создавать контроллеры и Resolve’ить зависимости.

## Решение

Для решения выше поставленной проблемы мы можем реализовать собственный IHttpControllerActivator, в методе Create которого будем получать созданный ранее контейнер и уже в рамках него Resolve’ить зависимости:

```csharp
public class ControllerActivator : IHttpControllerActivator
{
    public IHttpController Create(
        HttpRequestMessage request,
        HttpControllerDescriptor controllerDescriptor,
        Type controllerType
    )
    {
        // Get container that we set to OwinContext using common key
        var container = request.GetOwinContext().Get<IUnityContainer>(HttpApplicationKey.OwinPerRequestUnityContainerKey);

        // Resolve requested IHttpController using current container
        // prevent DefaultControllerActivator's behaviour of creating child containers
        var controller = (IHttpController)container.Resolve(controllerType);

        // Dispose container that would dispose each of container's registered service
        // Two ways of disposing container:
        // 1. At UnityContainerPerRequestMiddleware, after owin pipeline finished (WebAPI is just a part of pipeline)
        // 2. Here, after web api pipeline finished (if you do not use container at other middlewares) (uncomment next line)
        // request.RegisterForDispose(new Release(() => container.Dispose()));

        return controller;
    }
}
```

Для того, чтобы использовать его в Web API всё что нам остаётся, это заменить стандартный HttpControllerActivator в конфигурации:

```csharp
var config = new HttpConfiguration {
    DependencyResolver = new UnityDependencyResolver(container)
};

// Use our own IHttpControllerActivator implementation
// (to prevent DefaultControllerActivator's behaviour of creating child containers per request)
config.Services.Replace(typeof(IHttpControllerActivator), new ControllerActivator());

WebApiConfig.Register(config);
```

Таким образом, мы получаем следующий механизм работы с нашим единым контейнером:

Начало обработки запроса;

Создание дочернего контейнера от глобального;

```csharp
var childContainer = _container.CreateChildContainer();
```

Присваивание контейнера в Owin context:

```csharp
context.Set(HttpApplicationKey.OwinPerRequestUnityContainerKey, childContainer);
```

Использование контейнера в других Middleware’ах;

```csharp
var container = context.Get<IUnityContainer>(HttpApplicationKey.OwinPerRequestUnityContainerKey);
```

Использование контейнера в Web API;

Получение контроллера из Owin context:

```csharp
var container = request.GetOwinContext().Get<IUnityContainer>(HttpApplicationKey.OwinPerRequestUnityContainerKey);
```

Создание контроллера на основе этого контейнера:

```csharp
var controller = (IHttpController)container.Resolve(controllerType);
```

Уничтожение контейнера;

```csharp
childContainer.Dispose();
```

Завершение обработки запроса.

## Результат

Конфигурируем зависимости в соответствии с требуемыми нам их жизненными циклами:

```csharp
public static void RegisterTypes(IUnityContainer container)
{
    // ContainerControlledLifetimeManager - singleton's lifetime
    container.RegisterType<IAlwaysTheSame, AlwaysTheSame>(new ContainerControlledLifetimeManager());
    container.RegisterType<IAlwaysTheSame, AlwaysTheSame>(new ContainerControlledLifetimeManager());

    // HierarchicalLifetimeManager - container's lifetime
    container.RegisterType<ISameInARequest, SameInARequest>(new HierarchicalLifetimeManager());

    // TransientLifetimeManager (RegisterType's default) - no lifetime
    container.RegisterType<IAlwaysDifferent, AlwaysDifferent>(new TransientLifetimeManager());
}
```

1. ContainerControlledLifetimeManager - создание единственного экземпляра в рамках приложения;
2. HierarchicalLifetimeManager - создание единственного экземпляра в рамках контейнера (где мы добились того, что контейнер единый в рамках HTTP запроса);
3. TransientLifetimeManager - создание экземпляра при каждом обращении (Resolve).

![unity-per-request-middleware-result](/assets/images/posts/unity-per-request-middleware-result.png)

В изображении выше отображены GetHashCode’ы зависимостей в разрезе нескольких HTTP запросов, где:

1. AlwaysTheSame - singleton объект в рамках приложения;
2. SameInARequest - singleton объект в рамках запроса;
3. AlwaysDifferent - новый экземпляр для каждого Resolve.

Исходники доступны на [**GitHub**](https://github.com/FSou1/UnityPerRequestMiddleware).
