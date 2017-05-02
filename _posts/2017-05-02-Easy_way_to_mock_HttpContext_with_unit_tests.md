---
layout: post
title: Простой способ сделать mock HttpContext для UnitTest-ов
---

На днях реализовывал `PerHttpRequestLifeTimeManager` для своего небольшого [IoC-контейнера](https://github.com/FSou1/FsContainer) и поэтому хочу поделиться простым способом использовать HttpContext в покрытии тестами

Как уже наверняка стало понятно из заголовка, в основе реализации содержится экземпляр класса `HttpContext`, в статическом поле `HttpContext.Current` которого и содержится контекст текущего HTTP запроса. Для его эмуляции достаточно следующих строк кода:

```c#
HttpContext.Current = new HttpContext(
    new HttpRequest("", "http://any-uri.org", ""),
    new HttpResponse(new StringWriter())
);
```

Если требуется воспроизвести контекст с авторизованным пользователем, то можем добавить Principal:

```c#
HttpContext.Current.User = new GenericPrincipal(
    new GenericIdentity("username"),
    new string[0]
);
```

Вариант с пользователем после log out:

```c#
HttpContext.Current.User = new GenericPrincipal(
    new GenericIdentity(String.Empty),
    new string[0]
);
```

Решение выше не является моим и было найдено на просторах [stackoverflow](http://stackoverflow.com/questions/4379450/mock-httpcontext-current-in-test-init-method), однако, оказалось как никогда кстати и сильно мне помогло в моей задаче:

[TLDR: Github](https://github.com/FSou1/FsContainer/blob/master/Fs.Container.Web.Test/PerHttpRequestLifetimeManagerTest.cs)

В первом методе, т.к. используется один и тот же HttpContext, возвращаются одинаковые экземпляры зависимости, а во втором, напротив - различные.

```c#
[TestClass]
public class PerHttpRequestLifetimeManagerTest
{
    private FsContainer container;

    public PerHttpRequestLifetimeManagerTest()
    {
        container = new FsContainer();

        container
            .For<IMapper>()
            .Use<Mapper>(new PerHttpRequestLifetimeManager());
    }

    [TestMethod]
    public void TestPerSingleHttpRequestInstancesAlwaysSame()
    {
        // Arrange
        HttpContext.Current = new HttpContext(
            new HttpRequest("", "https://github.com/FSou1/FsContainer", ""),
            new HttpResponse(new StringWriter())
        );

        var controller = container.Resolve<Controller>();
        var firstMapper = container.Resolve<IMapper>();
        var secondMapper = container.Resolve<IMapper>();

        // Assert
        Assert.IsNotNull(controller);
        Assert.IsNotNull(controller.Mapper);
        Assert.IsNotNull(firstMapper);
        Assert.IsNotNull(secondMapper);
        Assert.AreSame(controller.Mapper, firstMapper);
        Assert.AreSame(controller.Mapper, secondMapper);
    }

    [TestMethod]
    public void TestPerMultipleHttpRequestInstancesAlwaysDifferent()
    {
        // Arrange
        HttpContext.Current = new HttpContext(
            new HttpRequest("", "https://github.com/FSou1/FsContainer", ""),
            new HttpResponse(new StringWriter())
        );
        var firstMapper = container.Resolve<IMapper>();

        HttpContext.Current = new HttpContext(
            new HttpRequest("", "https://github.com/FSou1", ""),
            new HttpResponse(new StringWriter())
        );
        var secondMapper = container.Resolve<IMapper>();

        // Assert
        Assert.IsNotNull(firstMapper);
        Assert.IsNotNull(secondMapper);
        Assert.AreNotSame(firstMapper, secondMapper);
    }
}
```