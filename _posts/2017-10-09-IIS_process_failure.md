---
title: IIS Process Failure - 502.5 request pipeline
tags: ASP.NET Core IIS
redirect_from: "/IIS_process_failure/"
---

С выходом ASP.NET Core 2.0 становится понятно, что новая платформа всё прочнее входит в наши реалии разработки, а значит пора начинать с ней знакомиться. Но начать мне бы хотелось не с разработки традиционных Hello world приложений и успешных сценариев, а наоборот, с отказа. И так уж случилось, что эта заметка станет в большей степени некоторым research'ем поведения IIS'а и лишь вскользь затронет [AspNetCoreModule](https://github.com/aspnet/AspNetCoreModule).

Если прямо сейчас создать новый Web Application в Visual Studio 17 (15.3) для платформы ASP.NET Core 2.0, то в файле Program.cs можно будет увидеть следующее:

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        BuildWebHost(args).Run();
    }

    public static IWebHost BuildWebHost(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseStartup<Startup>()
            .Build();
}
```

Как мы уже знаем из многочисленных видео с конференции [.NET Conf 2017](https://ingeno.io/2017/09/net-conf-2017-videos/) данный код является отправной точкой и каркасом нашего приложения.

Признаться, первоначальной темой для заметки я хотел выбрать обзор того, что же происходит в данных строчках, однако мысль "а что если" предопределила дальнейшее повествование. И заключается она в том, чтобы закомментировать содержимое метода `Main` (который с недавних пор может быть помечен ключевым словом `async`) и посмотреть что будет.

Сказано- сделано. И вот к чему это нас привело:

![iis_request_failure](/assets/images/posts/iis_request_failure.png)

Вопрос который возник у меня,- откуда берётся содержимое этой ошибки, кто ответственен за её возврат и что же происходит за ширмой. Очевидно, что начать следует с запущенного процесса IIS и чтобы как-то его идентифицировать, направляемся в свойства проекта.

![aspnetcore_project](/assets/images/posts/aspnetcore_project_settings.png)

Из информации о запуске проекта мы видим, что наше приложение использует IIS Express (и к слову, запускается так же копия приложения для HTTPS, слушаюшая 44319 порт).

![iis_applications](/assets/images/posts/iis_applications.png)

Корневым каталогом для нашего веб-сервера является путь `C:\Users\<Username>\Documents\IISExpress`.

Для начала убедимся что IIS действительно обрабатывал запрос и вернул соответствующий код ошибки. Для этого смотрим содержимое папки `logs`:

![iis_logs](/assets/images/posts/iis_logs.png)

Так и есть- содержимое лога красноричиво нам заявляет, что, дескать, были запросы по адресу `/` и код ответа по ним был возвращен `502 5`.

Далее рассмотрим содержимое папки `TraceLogFiles`.

![iis_logs](/assets/images/posts/iis_trace_events.png)

В моём случае в ней находятся 2 файла, каждый из которых содержит подробную информацию о событиях связанных с конкретным HTTP запросом. Я сперва был несколько удивлён почему их два, но присмотревшись всё оказалось довольно просто- наличие второго является следствием обращения к `favicon.ico`. Проанализировав содержимое становится очевидно, что наш запрос в конце концов поступает на обработку в модуль [AspNetCoreModule](https://github.com/aspnet/AspNetCoreModule):

![iis_trace_events_aspnetcore](/assets/images/posts/iis_trace_events_aspnetcore.png)

При поиске по исходникам этого модуля находим как [содержимое ошибки](https://github.com/aspnet/AspNetCoreModule/blob/746f578c3c01f0141479d5d9f31095ab6aba1935/src/AspNetCore/Inc/applicationmanager.h#L128) так и [код его добавления](https://github.com/aspnet/AspNetCoreModule/blob/746f578c3c01f0141479d5d9f31095ab6aba1935/src/AspNetCore/Src/forwardinghandler.cxx#L1399-L1413) в тело ответа от сервера:

```cpp
if (SUCCEEDED(pApplicationManager->Get502ErrorPage(&pDataChunk)))
{
    if (FAILED(hr = pResponse->WriteEntityChunkByReference(pDataChunk)))
    {
        goto Finished;
    }
    pResponse->SetStatus(502, "Bad Gateway", 5, hr);
    pResponse->SetHeader("Content-Type",
        "text/html",
        (USHORT)strlen("text/html"),
        FALSE
    );
    goto Finished;
}
```
