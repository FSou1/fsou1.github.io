---
title: What is Azure App Configuration?
tags: .NET Azure Architecture
redirect_from: "/What_is_azure_app_configuration/"
---

Hey guys! Today, I'd like to tell you about a new azure service called Azure App Configuration.

This service is intended to help developers manage their application and feature settings. It was announced in April 2019 and still in public preview mode. However, the documentation, samples and APIs are already available for use.

<iframe width="560" height="315" src="https://www.youtube.com/embed/vxyc_IOZnGA" frameborder="0" class="center-image" allow="autoplay; encrypted-media" allowfullscreen></iframe>

## Why use Azure App Configuration?

Usually, creating scalable and robust cloud-based applications is not an easy task:

![azure-app-configuration](/assets/images/posts/1-assets-why-use-app-configuration.png){: .center-image }

Especially considering, that applications often run on multiple virtual machines or even in multiple regions:

![azure-app-configuration](/assets/images/posts/2-assets-why-use-app-configuration.png){: .center-image }

And one of the well-proven practice in this case, is to separate configuration from code:

![azure-app-configuration](/assets/images/posts/3-assets-why-use-app-configuration.png){: .center-image }

🔥 And that's what App Configuration does best - centrally manages application settings.

## How it works

First of all, we need to create an instance of Azure App Configuration. After the deployment is finished, we can navigate to the service. Now it's time to realize what data of our application we'd like to store and manage. So let's add some key value.

I'll add a message with the key `MyApp:Settings:Message` and the value `Hello from App Configuration!`. l leave "Label" and "Content-Type" fields empty now. Once it's saved, we can see the added key value pair.

![azure-app-configuration](/assets/images/posts/4-assets-why-use-app-configuration.png)

❗ One point I'd like to mention here, is that even though App Configuration offers complete data encryptions, and encrypts all key values it holds and network communication, you should not store secrets here. Azure Key Vault is still the best place for storing application secrets.

Anyway, let's go back to our application. I have created a new ASP .NET Core application and now it's time to connect our app configuration instance. In order to do that, we need to:

- Add a reference to a nuget package `Microsoft.Azure.AppConfiguration.AspNetCore`;

- Update the `CreateWebHostBuilder` method to use App Configuration:

```
public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((hostingContext, config) =>
        {
            var settings = config.Build();
            config.AddAzureAppConfiguration(settings["ConnectionStrings:AppConfig"]);
        })
        .UseStartup<Startup>();
```

- Get a connection string to our service from the "Access keys" tab and add it in the local `appsettings.json` file:

```
{
  "ConnectionStrings": {
    "AppConfig": "Endpoint=https://my-app-configuration.azconfig.io;Id=1-l9-s0:QR0x"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

- And the last but not least, is to inject `IConfiguration` dependency into our controller and obtain the configuration value, that we've recently added:

```
private readonly IConfiguration configuration;

public ValuesController(IConfiguration configuration)
{
    this.configuration = configuration;
}

// GET api/values
[HttpGet]
public ActionResult<IEnumerable<string>> Get()
{
    string message = this.configuration["MyApp:Settings:Message"];

    return new string[] { message };
}
```

Once it's done, we can start the application.. and ensure, that everything works,- the returned message was successfully obtained from our App Configuration service:

![azure-app-configuration](/assets/images/posts/5-assets-why-use-app-configuration.png)

## Summary

Even though this service is useful in various number of use-cases, I believe microservices, serverless applications and CI/CD pipelines will benefit the most.

Just to summarize, let's take a look what does App Configuration actually offer:

- A fully managed service that can be set up in minutes;
- Flexible key representations and mappings;
- Tagging with labels;
- Point-in-time replay of settings;
- Dedicated UI for feature flag management;
- Integration with Azure-managed identities;
- Complete data encryptions;
- Integration with popular frameworks.

I'm hoping you will agree with me that all of these features right out of the box- that's really cool!

Thank you for reading and see you next time!

Reference:

1. [Documentation](https://docs.microsoft.com/en-us/azure/azure-app-configuration/);
2. [FAQ](https://docs.microsoft.com/en-us/azure/azure-app-configuration/faq).
