---
title: SignalR Core- how to start creating a currency broadcaster sample
tags: .NET SignalR Core
redirect_from: "/SignalR_core_how_to/"
---

Today I'd like to take a look at a new version of SignalR package (to be honest, it's official name is `Microsoft.AspNetCore.Signalr`) and implement a currency broadcaster sample.

Currently (22 Jan 2018) the package is still in the alpha version, but the final release is going to be published very soon with the official AspNetCore release.

## Default startup

Let's start with adding the nuget package to our newly created project with the usage of CLI:

`Install-Package Microsoft.AspNetCore.SignalR -Version 1.0.0-alpha2-final`

Or a nuget package manager:

![signalr_nuget](/assets/images/posts/signalr_nuget.png)

After that we have to tell to our application that we are going to use the recently added SignalR package. First of all, we need to update _Startup.cs_'s `ConfigureServices` method like this:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSignalR();
    services.AddMvc();
}
```

Let's take a look at the internal details of the `AddSignalR` method. It's just an extension method over the `IServiceCollection` interface and is used for a default signalr services registration:

```csharp
services.Configure<HubOptions>(configure);
services.AddSockets();
services.AddSingleton(typeof (HubLifetimeManager<>), typeof (DefaultHubLifetimeManager<>));
services.AddSingleton(typeof (IHubProtocolResolver), typeof (DefaultHubProtocolResolver));
services.AddSingleton(typeof (IHubContext<>), typeof (HubContext<>));
services.AddSingleton(typeof (IHubContext<,>), typeof (HubContext<,>));
services.AddSingleton(typeof (HubEndPoint<>), typeof (HubEndPoint<>));
services.AddScoped(typeof (IHubActivator<>), typeof (DefaultHubActivator<>));
services.AddAuthorization();
```

Then we have to declare a new class which is derived from the `Hub` one. With this sample the only purpose is to handle the particular endpoint request, but it could also contain a logic, which should be accessible from a frontend (or fired due to `OnConnected` and `OnDisconnected` events). I've named it as `CurrencyHub` and left it empty:

```csharp
public class CurrencyHub : Hub
{

}
```

Last but not least we need to update the method `Configure` with the `UseSignalR` method call:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    // ..

    app.UseSignalR(routes => {
        routes.MapHub<CurrencyHub>("currency");
    });

    // ..
}
```

We use the `UseSignalR` method for configuring our routes table: every `/currency` http request will be handled by our `CurrencyHub` class.

If everything has been done in a proper way then the next output should appear:

![signalr_first_currency](/assets/images/posts/signalr_first_currency.png)

Now your backend part of the application is fully configured and could be used as a base for a futher implementation. But as I've mentioned before, I'd like to implement a working sample so we are only half way there ;)

## Sample implementation

I'd like to implement a pretty straightforward stock currencies display. This display should contain information about a set of currencies and this display should have the ability to update in real-time (once per second) for every connected user simultaneously.

The sample contains three main parts:

- Broadcaster as a component, which should fetch or listen to information about currency updates;
- Notifier as a component, which should notify all the active connections with up-to-date data;
- Display table as a currency information view;

I've implemented a broadcaster with the usage of `IHostedService` ([read more](https://blogs.msdn.microsoft.com/cesardelatorre/2017/11/18/implementing-background-tasks-in-microservices-with-ihostedservice-and-the-backgroundservice-class-net-core-2-x/)). It's a pretty usefull interface with an amazing benefit, like hosting your component as an AspNetCore background worker:

```csharp
public class CurrencyListenerService : IHostedService, IDisposable
{
    //..

    public async Task StartAsync(CancellationToken cts)
    {
        while (!cts.IsCancellationRequested)
        {
            var currencies = await FetchCurrencyUpdates();

            await Broadcast(currencies);

            await Task.Delay(_delay, cts);
        }
    }

    private Task<Dictionary<string, double>> FetchCurrencyUpdates()
    {
        var data = new Dictionary<string, double>
        {
            {"EUR", Get(69) }, {"USD", Get(56) }, {"GBR", Get(79) },
            {"INR", Get(1) }, {"CAD", Get(45) }, {"MAD", Get(6) },
            {"AUD", Get(45) }, {"TRY", Get(15) }, {"AZN", Get(33) },
        };

        return Task.FromResult(data);
    }

    //..
}
```

The broadcaster part of this service has to be implemented with the usage of `HubLifetimeManager<THub>`. Based on a generic type `THub` this class contains information about all the active connections of the particular hub and has an opportunity to notify them throught a function call, which is declared at frontend:

```csharp
private async Task Broadcast(Dictionary<string, double> data)
{
    await _hubManager.InvokeAllAsync("currenciesUpdated",
        new object[] { data });
}
```

To host this service we need to register it at `IServiceCollection` like this:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<IHostedService, CurrencyListenerService>();

    //..
}
```

As I've already mentioned before we need to declare a function at frontend, which could be called from a backend side for a notifier purpose (let it be `currenciesUpdated` function).

Recently i've taken a look at [Vue.js](https://vuejs.org/) and found it pretty nice and straightforward. Bellow you can see several parts of code with the usage of this library:

```javascript
<script>
    var app = new Vue({
        el: "#app",
        data: {
            currencies: null,
            changes: {}
        },
        methods: {
            hubConnect: function() {
                var connection = new signalR.HubConnection('/currency');

                connection.on('currenciesUpdated',
                    function(data) {
                        this.currencies = data;
                    }.bind(this));

                connection.onclose(e => {
                    if (e) {
                        console.log('Connection closed with error: ' + e, 'red');
                    } else {
                        console.log('Disconnected', 'green');
                    }
                });

                connection.start();
            }
        },
        watch: {
            currencies: function (newVal, oldVal) {
                if (!newVal || !oldVal) return;
                for (var key in newVal) {
                    if (newVal.hasOwnProperty(key)) {
                        this.changes[key] = newVal[key] > oldVal[key];
                    }
                }
            }
        },
        mounted: function() {
            this.hubConnect();
        }
    });
</script>
```

And a layout:

```html
<table class="table table-bordered table-striped" v-if="currencies">
  <tr>
    <th></th>
    <th v-for="(value, key) in currencies">{ key.toUpperCase() }</th>
  </tr>

  <tr>
    <th><img src="http://www.xe.com/themes/xe/images/flags/rub.png" /> RUB</th>
    <td
      v-for="(value, key) in currencies"
      v-bind:class="{ 'up': changes[key], 'down': !changes[key] }"
    >
      {{ value }}
    </td>
  </tr>
</table>
```

Looks really nice, does not it?

It doesn't really matter which frontend framework you are going to use. The only thing that is **really important** is not to forget to append a reference to the SignalR library, because otherwise a console output will give you a headache:

```html
<script src="~/lib/signalr-client-1.0.0-alpha2-final.min.js"></script>
```

For now it's still in alpha (and more than 158 issues are still open at [GitHub repository](https://github.com/aspnet/SignalR)), but npm module is already available: ([@aspnet/signalr-client](https://www.npmjs.com/package/@aspnet/signalr-client)):

> **Usage**:
>
> To use the client in a browser, copy *.js files from the dist/browser folder to your script folder include on your page using the *script\* tag.
>
> **Example**:
>
> ```javascript
> let connection = new signalR.HubConnection("/chat");
>
> connection.on("send", (data) => {
>   console.log(data);
> });
>
> connection.start().then(() => connection.invoke("send", "Hello"));
> ```

And finally here is a demo of the result:

![chsarp_71_ep](/assets/images/posts/currency-signalr-result.gif)

Thanks for reading, I really hope you found it usefull and feel free to comment!

Summary:

1. [Github repository](https://github.com/FSou1/CoreSignalR.Sample);
2. [Announcing SignalR (alpha) for ASP.NET Core 2.0](https://blogs.msdn.microsoft.com/webdev/2017/09/14/announcing-signalr-for-asp-net-core-2-0/);
3. [Implementing background tasks in .NET Core 2.x webapps or microservices with IHostedService and the BackgroundService class](https://blogs.msdn.microsoft.com/cesardelatorre/2017/11/18/implementing-background-tasks-in-microservices-with-ihostedservice-and-the-backgroundservice-class-net-core-2-x/);
