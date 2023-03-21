---
title: Простейшее расширение для перехвата запроса в owin pipeline
tags: .NET Middleware
redirect_from: "/Simple_uptime_monitoring_middleware/"
---

В сегодняшней небольшой заметке хотел бы поделиться очень простым расширением, которое, например, можно использовать для отслеживания uptime приложения (так же доступно на [github gist](https://gist.github.com/FSou1/c6f61a9c0591e0ff59826b92485c2d95)).

Owin middleware:

```csharp
public class UptimeMonitoringMiddleware
{
    public UptimeMonitoringMiddleware(
        Func<IDictionary<string, object>, Task> next, string path)
    {
        _next = next;
        _path = path;
    }

    public async Task Invoke(IDictionary<string, object> environment)
    {
        var context = new OwinContext(environment);

        if (context.Request.Path == PathString.FromUriComponent(_path ?? HttpPropertyKeys.UptimeMiddlewarePath))
        {
            await context.Response.WriteAsync("Uptime status: OK");
            return;
        }

        await _next(environment);
    }

    private readonly Func<IDictionary<string, object>, Task> _next;
    private readonly string _path;
}

public static class UptimeMiddlewareExtension
{
    public static IAppBuilder UseUptimeMonitoringMiddleware(this IAppBuilder app, string path = null)
    {
        app.Use(typeof(UptimeMonitoringMiddleware), path);
        return app;
    }
}
```

Константа:

```csharp
public static class HttpPropertyKeys
{
    public static readonly string UptimeMiddlewarePath = "/uptime";
}
```
