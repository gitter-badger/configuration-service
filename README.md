# Configuration Service

[![Build](https://github.com/jamespratt/configuration-service/workflows/release/badge.svg)](https://github.com/jamespratt/configuration-service/actions?query=workflow%3Arelease)


|  Package  |Latest Release|
|:----------|:------------:|
|**ConfigurationService.Hosting**|[![NuGet Badge ConfigurationService.Hosting](https://buildstats.info/nuget/ConfigurationService.Hosting)](https://www.nuget.org/packages/ConfigurationService.Hosting)
|**ConfigurationService.Client**|[![NuGet Badge ConfigurationService.Client](https://buildstats.info/nuget/ConfigurationService.Client)](https://www.nuget.org/packages/ConfigurationService.Client)

## About Configuration Service

Configuration Service is a remote configuration service for .NET Core.  Configuration for fleets of applications, services, and containerized micro-services can be updated immediately without the need to redeploy or restart. Configuration Service uses a client/server pub/sub architecture to notify subscribed clients of configuration changes as they happen.  Configuration can be injected using the standard options pattern with `IOptions`, `IOptionsMonitor` or `IOptionsSnapshot`.

Configuration Service currently supports hosting configuration with either git or a file system and supports publishing changes with Redis or RabbitMQ publish/subscribe.  File types supported are .json, .yaml, .xml and .ini.

[![Configuration Service Diagram](https://github.com/jamespratt/configuration-service/blob/master/images/configuration-service.png)](#about-configuration-service)

## Features
* RESTful HTTP based API for external configuration.
* Server easily integrates into an ASP.NET Core application.
* Client easily integrates into any .NET Standard 2.0 application using the standard `ConfigurationBuilder` pattern.
* Client encapsulates real-time configuration updates.
* Support for git and file system based storage.
* Support for .json, .yaml, .xml and .ini configuration files.
* Inject configuration with `IOptionsMonitor` or `IOptionsSnapshot` to access configuration changes.

## Installing with NuGet

The easiest way to install Configuration Service is with [NuGet](https://www.nuget.org/packages/ConfigurationService.Hosting/).

In Visual Studio's [Package Manager Console](http://docs.nuget.org/docs/start-here/using-the-package-manager-console),
enter the following command:

Server:

    Install-Package ConfigurationService.Hosting
    
Client:

    Install-Package ConfigurationService.Client
    
## Adding the Configuration Service Host
The Configuration Service host middleware can be added to the service collection of an existing ASP.NET Core application.  The following example configures a git storage provider with a Redis publisher.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();

    services.AddConfigurationService()
        .AddGitProvider(c =>
        {
            c.RepositoryUrl = "https://example.com/my-repo/configuration.git";
            c.Username = "username";
            c.Password = "password";
            c.LocalPath = "C:/config";
        })
        .AddRedisPublisher("localhost:6379");
}
```

In Startup.Configure, call `MapConfigurationService` on the endpoint builder. The default pattern is "/configuration".

```csharp
app.UseEndpoints(endpoints =>
{
    endpoints.MapConfigurationService();
});
```

The configured host will expose two API endpoints:
* `configuration/` - Lists all files at the configured provider.
* `configuration/{filename}` - Retrieves the contents of the specified file.

#### Git Provider Options
|  Property  | Description |
|:-----------|:------------|
|RepositoryUrl|URI for the remote repository.|
|Username|Username for authentication.|
|Password|Password for authentication.|
|Branch|The name of the branch to checkout. When unspecified the remote's default branch will be used instead.|
|LocalPath|Local path to clone into.|
|SearchPattern|The search string to use as a filter against the names of files. Defaults to no filter (*).|
|PollingInterval|The interval to check for remote changes. Defaults to 60 seconds.|

```csharp
services.AddConfigurationService()
    .AddGitProvider(c =>
    {
        c.RepositoryUrl = "https://example.com/my-repo/my-repo.git";
        c.Username = "username";
        c.Password = "password";
        c.Branch = "master";
        c.LocalPath = "C:/config";
        c.SearchPattern = ".*json";
        c.PollingInterval = TimeSpan.FromSeconds(60);
    }
    ...
```

#### File System Provider Options
|  Property  | Description |
|:-----------|:------------|
|Path|Path to the configuration files.|
|SearchPattern|The search string to use as a filter against the names of files. Defaults to no filter (*).|
|IncludeSubdirectories|Includes the current directory and all its subdirectories. Defaults to `false`.|
|Username|Username for authentication.|
|Password|Password for authentication.|
|Domain|Domain for authentication.|

```csharp
services.AddConfigurationService()
    .AddFileSystemProvider(c => 
    {
        c.Path = "C:/config";
        c.SearchPattern = "*.json";
        c.IncludeSubdirectories = true;
    })
    ...
```

#### Custom Storage Providers and Publishers
Custom implementations of storage providers and publishers can be added by implementing the `IProvider` and `IPublisher` interfaces and calling the appropriate extension methods on AddConfigurationService:

```csharp
services.AddConfigurationService()
    .AddCustomProvider(new CustomStorageProvider())
    .AddCustomPublisher(new CustomPublisher());
```

## Adding the Configuration Service Client
The Configuration Service client can be configured by adding `AddRemoteSource` to the standard configuration builder. In the following example, remote json configuration is added and a Redis endpoint is specified for configuration change subscription.  Local configuration can be read for settings for the remote source by using multiple instances of the configuration. 

```csharp
var loggerFactory = LoggerFactory.Create(builder =>
{
    builder.AddConsole();
});

IConfiguration configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .Build();

configuration = new ConfigurationBuilder()
    .AddConfiguration(configuration)
    .AddRemoteSource(s => 
    {
        s.ConfigurationName = "text.json";
        s.ConfigurationServiceUri = "http://localhost:5000/configuration/";
        s.Subscriber = () => new RedisSubscriber("localhost:6379");
        s.Optional = false;
        s.ReloadOnChange = true;
        s.LoggerFactory = loggerFactory;
    })
    .Build();
```

#### Configuration Soruce Options
|  Property  | Description |
|:-----------|:------------|
|ConfigurationName|Path or name of the configuration file relative to the configuration provider. This value should match the value specified in the list returned by the `configuration/` endpoint.|
|ConfigurationServiceUri|Configuration service endpoint.|
|Optional|Determines if loading the file is optional.|
|ReloadOnChange|Determines whether the source will be loaded if the underlying file changes.|
|HttpMessageHandler|The optional `HttpMessageHandler` for the `HttpClient`.|
|RequestTimeout|The timeout for the `HttpClient` request to the configuration server. Defaults to 60 seconds.|
|Parser|The type used to parse the remote configuration file. The client will attempt to resolve this from the file extension of `ConfigurationName` if not specified.<br /><br />Supported Types: <ul><li>`JsonConfigurationFileParser`</li><li>`YamlConfigurationFileParser`</li><li>`XmlConfigurationFileParser`</li><li>`IniConfigurationFileParser`</li></ul>|
|Subscriber|Delegate to create the type of `ISubscriber` used to subscribe to published configuration messages.|
|LoggerFactory|The type used to configure the logging system and create instances of `ILogger`. Defaults to `NullLoggerFactory`.|

## Samples
Samples of both host and client implementations can be viewed at [Samples](https://github.com/jamespratt/configuration-service/tree/master/samples).

[![Build history](https://buildstats.info/github/chart/jamespratt/configuration-service)](https://github.com/jamespratt/configuration-service/actions?query=workflow%3Arelease) [![Join the chat at https://gitter.im/configuration-service/community](https://badges.gitter.im/configuration-service/community.svg)](https://gitter.im/configuration-service/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
