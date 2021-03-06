---
title: Configuration in ASP.NET Core
author: rick-anderson
description: Learn how to use the Configuration API to configure an ASP.NET Core app from multiple sources.
keywords: ASP.NET Core,configuration,JSON,config
ms.author: riande
manager: wpickett
ms.date: 6/24/2017
ms.topic: article
ms.assetid: b3a5984d-e172-42eb-8a48-547e4acb6806
ms.technology: aspnet
ms.prod: asp.net-core
uid: fundamentals/configuration
---
# Configuration in ASP.NET Core

[Rick Anderson](https://twitter.com/RickAndMSFT), [Mark Michaelis](http://intellitect.com/author/mark-michaelis/), [Steve Smith](https://ardalis.com/), [Daniel Roth](https://github.com/danroth27), and [Luke Latham](https://github.com/guardrex)

The Configuration API provides a way of configuring an app based on a list of name-value pairs. Configuration is read at runtime from multiple sources. The name-value pairs can be grouped into a multi-level hierarchy. There are configuration providers for:

* File formats (INI, JSON, and XML)
* Command-line arguments
* Environment variables
* In-memory .NET objects
* An encrypted user store
* [Azure Key Vault](xref:security/key-vault-configuration)
* Custom providers, which you install or create

Each configuration value maps to a string key. There's built-in binding support to deserialize settings into a custom [POCO](https://wikipedia.org/wiki/Plain_Old_CLR_Object) object (a simple .NET class with properties).

[View or download sample code](https://github.com/aspnet/docs/tree/master/aspnetcore/fundamentals/configuration/sample) ([how to download](xref:tutorials/index#how-to-download-a-sample))

## Simple configuration

The following console app uses the JSON configuration provider:

[!code-csharp[Main](configuration/sample/ConfigJson/Program.cs)]

The app reads and displays the following configuration settings:

[!code-json[Main](configuration/sample/ConfigJson/appsettings.json)]

Configuration consists of a hierarchical list of name-value pairs in which the nodes are separated by a colon. To retrieve a value, access the `Configuration` indexer with the corresponding item's key:

```csharp
Console.WriteLine($"option1 = {Configuration["subsection:suboption1"]}");
```

To work with arrays in JSON-formatted configuration sources, use a array index as part of the colon-separated string. The following example gets the name of the first item in the preceding `wizards` array:

```csharp
Console.Write($"{Configuration["wizards:0:Name"]}, ");
```

Name-value pairs written to the built in `Configuration` providers are **not** persisted, however, you can create a custom provider that saves values. See [custom configuration provider](xref:fundamentals/configuration#custom-config-providers).

The preceding sample uses the configuration indexer to read values. To access configuration outside of `Startup`, use the [options pattern](xref:fundamentals/configuration#options-config-objects). The *options pattern* is shown later in this article.

It's typical to have different configuration settings for different environments, for example, development, test, and production. The `CreateDefaultBuilder` extension method in an ASP.NET Core 2.x app (or using `AddJsonFile` and `AddEnvironmentVariables` directly in an ASP.NET Core 1.x app) adds configuration providers for reading JSON files and system configuration sources:

* *appsettings.json*
* *appsettings.\<EnvironmentName>.json*
* environment variables

See [AddJsonFile](https://docs.microsoft.com/aspnet/core/api/microsoft.extensions.configuration.jsonconfigurationextensions) for an explanation of the parameters. `reloadOnChange` is only supported in ASP.NET Core 1.1 and higher. 

Configuration sources are read in the order they are specified. In the code above, the environment variables are read last. Any configuration values set through the environment would replace those set in the two previous providers.

The environment is typically set to one of `Development`, `Staging`, or `Production`. See [Working with Multiple Environments](environments.md) for more information.

Configuration considerations:

* `IOptionsSnapshot` can reload configuration data when it changes. Use `IOptionsSnapshot` if you need to reload configuration data.  See [IOptionsSnapshot](#ioptionssnapshot) for more information.
* Configuration keys are case insensitive.
* A best practice is to specify environment variables last, so that the local environment can override anything set in deployed configuration files.
* **Never** store passwords or other sensitive data in configuration provider code or in plain text configuration files. Don't use production secrets in your development or test environments. Instead, specify secrets outside the project tree, so they cannot be accidentally committed into your repository. Learn more about [Working with Multiple Environments](environments.md) and managing [safe storage of app secrets during development](../security/app-secrets.md).
* If `:` cannot be used in environment variables in your system,  replace `:`  with `__` (double underscore).

<a name="options-config-objects"></a>

## Using Options and configuration objects

The options pattern uses custom options classes to represent a group of related settings. We recommended that you create decoupled classes for each feature within your app. Decoupled classes follow:

* The [Interface Segregation Principle (ISP)](http://deviq.com/interface-segregation-principle/) : Classes depend only on the configuration settings they use.
* [Separation of Concerns](http://deviq.com/separation-of-concerns/) : Settings for different parts of your app are not dependent or coupled with one another.

The options class must be non-abstract with a public parameterless constructor. For example:

[!code-csharp[Main](configuration/sample/UsingOptions/Models/MyOptions.cs)]

<a name="options-example"></a>

In the following code, the JSON configuration provider is enabled. The `MyOptions` class is added to the service container and bound to configuration.

[!code-csharp[Main](configuration/sample/UsingOptions/Startup.cs?name=snippet1&highlight=8,20-21)]

The following [controller](../mvc/controllers/index.md)  uses [constructor Dependency Injection](xref:fundamentals/dependency-injection#what-is-dependency-injection) on [`IOptions<TOptions>`](https://docs.microsoft.com/aspnet/core/api/microsoft.extensions.options.ioptions-1) to access settings:

[!code-csharp[Main](configuration/sample/UsingOptions/Controllers/HomeController.cs?name=snippet1)]

With the following *appsettings.json* file:

[!code-json[Main](configuration/sample/UsingOptions/appsettings1.json)]

The `HomeController.Index` method returns `option1 = value1_from_json, option2 = 2`.

Typical apps won't bind the entire configuration to a single options file. Later on I'll show how to use `GetSection` to bind to a section.

In the following code, a second `IConfigureOptions<TOptions>` service is added to the service container. It uses a delegate to configure the binding with `MyOptions`.

[!code-csharp[Main](configuration/sample/UsingOptions/Startup2.cs?name=snippet1&highlight=9-13)]

You can add multiple configuration providers. Configuration providers are available in NuGet packages. They are applied in order they are registered.

Each call to `Configure<TOptions>` adds an `IConfigureOptions<TOptions>` service to the service container. In the preceding example, the values of `Option1` and `Option2` are both specified in *appsettings.json* -- but the value of `Option1` is overridden by the configured delegate. 

When more than one configuration service is enabled, the last configuration source specified "wins" (sets the configuration value). In the preceding code, the `HomeController.Index` method returns `option1 = value1_from_action, option2 = 2`.

When you bind options to configuration, each property in your options type is bound to a configuration key of the form `property[:sub-property:]`. For example, the `MyOptions.Option1` property is bound to the key `Option1`, which is read from the `option1` property in *appsettings.json*. A sub-property sample is shown later in this article.

In the following code, a third `IConfigureOptions<TOptions>` service is added to the service container. It binds `MySubOptions` to the section `subsection` of the *appsettings.json* file:

[!code-csharp[Main](configuration/sample/UsingOptions/Startup3.cs?name=snippet1&highlight=16-17)]

Note: This extension method requires the `Microsoft.Extensions.Options.ConfigurationExtensions` NuGet package.

Using the following *appsettings.json* file:

[!code-json[Main](configuration/sample/UsingOptions/appsettings.json)]

The `MySubOptions` class:

[!code-csharp[Main](configuration/sample/UsingOptions/Models/MySubOptions.cs?name=snippet1)]

With the following `Controller`:

[!code-csharp[Main](configuration/sample/UsingOptions/Controllers/HomeController2.cs?name=snippet1)]

`subOption1 = subvalue1_from_json, subOption2 = 200` is returned.

You can also supply options in a view model or inject `IOptions<TOptions>` directly into a view:

[!code-html[Main](configuration/sample/UsingOptions/Views/Home/Index.cshtml?highlight=3-4,16-17,20-21)]

<a name="in-memory-provider"></a>

## IOptionsSnapshot

*Requires ASP.NET Core 1.1 or higher.*

`IOptionsSnapshot` supports reloading configuration data when the configuration file has changed. It has minimal overhead. Using `IOptionsSnapshot` with `reloadOnChange: true`, the options are bound to `IConfiguration` and reloaded when changed.

The following sample demonstrates how a new `IOptionsSnapshot` is created after *config.json* changes. Requests to the server will return the same time when *config.json* has **not** changed. The first request after *config.json* changes will show a new time.

[!code-csharp[Main](configuration/sample/IOptionsSnapshot2/Startup.cs?name=snippet1&highlight=1-9,13-18,32,33,52,53)]

The following image shows the server output:

![browser image showing "Last Updated: 11/22/2016 4:43 PM"](configuration/_static/first.png)

Refreshing the browser doesn't change the message value or time displayed (when *config.json* has not changed).

Change and save the  *config.json* and then refresh the browser:

![browser image showing "Last Updated to,e: 11/22/2016 4:53 PM"](configuration/_static/change.png)

## In-memory provider and binding to a POCO class

The following sample shows how to use the in-memory provider and bind to a class:

[!code-csharp[Main](configuration/sample/InMemory/Program.cs)]

Configuration values are returned as strings, but binding enables the construction of objects. Binding allows you to retrieve POCO objects or even entire object graphs. The following sample shows how to bind to `MyWindow` and use the options pattern with a ASP.NET Core MVC app:

[!code-csharp[Main](configuration/sample/WebConfigBind/MyWindow.cs)]

[!code-json[Main](configuration/sample/WebConfigBind/appsettings.json)]

Bind the custom class in `ConfigureServices` when building the host:

[!code-csharp[Main](configuration/sample/WebConfigBind/Program.cs?name=snippet1&highlight=3-4)]

Display the settings from the `HomeController`:

[!code-csharp[Main](configuration/sample/WebConfigBind/Controllers/HomeController.cs)]

### GetValue

The following sample demonstrates the [GetValue<T>](https://docs.microsoft.com/aspnet/core/api/microsoft.extensions.configuration.configurationbinder#Microsoft_Extensions_Configuration_ConfigurationBinder_GetValue_Microsoft_Extensions_Configuration_IConfiguration_System_Type_System_String_System_Object_) extension method:

[!code-csharp[Main](configuration/sample/InMemoryGetValue/Program.cs?highlight=27-29)]

The ConfigurationBinder's `GetValue<T>` method allows you to specify a default value (80 in the sample). `GetValue<T>` is for simple scenarios and does not bind to entire sections. `GetValue<T>` gets scalar values from `GetSection(key).Value` converted to a specific type.

## Binding to an object graph

You can recursively bind to each object in a class. Consider the following `AppOptions` class:

[!code-csharp[Main](configuration/sample/ObjectGraph/AppOptions.cs)]

The following sample binds to the `AppOptions` class:

[!code-csharp[Main](configuration/sample/ObjectGraph/Program.cs?highlight=15-16)]

**ASP.NET Core 1.1** and higher can use  `Get<T>`, which works with entire sections. `Get<T>` can be more convenient than using `Bind`. The following code shows how to use `Get<T>` with the sample above:

```csharp
var appConfig = config.GetSection("App").Get<AppOptions>();
```

Using the following *appsettings.json* file:

[!code-json[Main](configuration/sample/ObjectGraph/appsettings.json)]

The program displays `Height 11`.

The following code can be used to unit test the configuration:

```csharp
[Fact]
public void CanBindObjectTree()
{
    var dict = new Dictionary<string, string>
        {
            {"App:Profile:Machine", "Rick"},
            {"App:Connection:Value", "connectionstring"},
            {"App:Window:Height", "11"},
            {"App:Window:Width", "11"}
        };
    var builder = new ConfigurationBuilder();
    builder.AddInMemoryCollection(dict);
    var config = builder.Build();

    var options = new AppOptions();
    config.GetSection("App").Bind(options);

    Assert.Equal("Rick", options.Profile.Machine);
    Assert.Equal(11, options.Window.Height);
    Assert.Equal(11, options.Window.Width);
    Assert.Equal("connectionstring", options.Connection.Value);
}
```

<a name="custom-config-providers"></a>

## Basic sample of Entity Framework custom provider

In this section, a basic configuration provider that reads name-value pairs from a database using EF is created. 

Define a `ConfigurationValue` entity for storing configuration values in the database:

[!code-csharp[Main](configuration/sample/CustomConfigurationProvider/ConfigurationValue.cs)]

Add a `ConfigurationContext` to store and access the configured values:

[!code-csharp[Main](configuration/sample/CustomConfigurationProvider/ConfigurationContext.cs?name=snippet1)]

Create an class that implements [IConfigurationSource](https://docs.microsoft.com/aspnet/core/api/microsoft.extensions.configuration.iconfigurationsource):

[!code-csharp[Main](configuration/sample/CustomConfigurationProvider/EntityFrameworkConfigurationSource.cs?highlight=7)]

Create the custom configuration provider by inheriting from [ConfigurationProvider](https://docs.microsoft.com/aspnet/core/api/microsoft.extensions.configuration.configurationprovider).  The configuration provider initializes the database when it's empty:

[!code-csharp[Main](configuration/sample/CustomConfigurationProvider/EntityFrameworkConfigurationProvider.cs?highlight=9,18-31,38-39)]

The highlighted values from the database ("value_from_ef_1" and "value_from_ef_2") are displayed when the sample is run.

You can add an `EFConfigSource` extension method for adding the configuration source:

[!code-csharp[Main](configuration/sample/CustomConfigurationProvider/EntityFrameworkExtensions.cs?highlight=12)]

The following code shows how to use the custom `EFConfigProvider`:

[!code-csharp[Main](configuration/sample/CustomConfigurationProvider/Program.cs?highlight=21-26)]

Note the sample adds the custom `EFConfigProvider` after the JSON provider, so any settings from the database will override settings from the *appsettings.json* file.

Using the following *appsettings.json* file:

[!code-json[Main](configuration/sample/CustomConfigurationProvider/appsettings.json)]

The following is displayed:

```console
key1=value_from_ef_1
key2=value_from_ef_2
key3=value_from_json_3
```

## CommandLine configuration provider

The [CommandLine configuration provider](/aspnet/core/api/microsoft.extensions.configuration.commandline.commandlineconfigurationprovider) receives command-line argument key-value pairs for configuration at runtime.

[View or download the CommandLine configuration sample](https://github.com/aspnet/docs/tree/master/aspnetcore/fundamentals/configuration/sample/CommandLine)

### Setting up the provider

# [Basic Configuration](#tab/basicconfiguration)

To activate command-line configuration, call the `AddCommandLine` extension method on an instance of [ConfigurationBuilder](/api/microsoft.extensions.configuration.configurationbuilder):

[!code-csharp[Main](configuration/sample_snapshot/CommandLine/Program.cs?highlight=18,21)]

Running the code, the following output is displayed:

```console
MachineName: MairaPC
Left: 1980
```

Passing argument key-value pairs on the command line changes the values of `Profile:MachineName` and `App:MainWindow:Left`:

```console
dotnet run Profile:MachineName=BartPC App:MainWindow:Left=1979
```

The console window displays:

```console
MachineName: BartPC
Left: 1979
```

To override configuration provided by other configuration providers with command-line configuration, call `AddCommandLine` last on `ConfigurationBuilder`:

[!code-csharp[Main](configuration/sample_snapshot/CommandLine/Program2.cs?range=11-16&highlight=1,5)]

# [ASP.NET Core 2.x](#tab/aspnetcore2x)

Typical ASP.NET Core 2.x apps use the static convenience method `CreateDefaultBuilder` to build the host:

[!code-csharp[Main](configuration/sample_snapshot/Program.cs?highlight=12)]

`CreateDefaultBuilder` loads optional configuration from *appsettings.json*, *appsettings.{Environment}.json*, [user secrets](xref:security/app-secrets) (in the `Development` environment), environment variables, and command-line arguments. The CommandLine configuration provider is called last. Calling the provider last allows the command-line arguments passed at runtime to override configuration set by the other configuration providers called earlier.

Note that for *appsettings* files that `reloadOnChange` is enabled. Command-line arguments are overridden if a matching configuration value in an *appsettings* file is changed after the app starts.

> [!NOTE]
> As an alternative to using the `CreateDefaultBuilder` method, creating a host using [WebHostBuilder](/dotnet/api/microsoft.aspnetcore.hosting.webhostbuilder) and manually building configuration with [ConfigurationBuilder](/api/microsoft.extensions.configuration.configurationbuilder) is supported in ASP.NET Core 2.x. See the ASP.NET Core 1.x tab for more information.

# [ASP.NET Core 1.x](#tab/aspnetcore1x)

Create a [ConfigurationBuilder](/api/microsoft.extensions.configuration.configurationbuilder) and call the `AddCommandLine` method to use the CommandLine configuration provider. Calling the provider last allows the command-line arguments passed at runtime to override configuration set by the other configuration providers called earlier. Apply the configuration to [WebHostBuilder](/dotnet/api/microsoft.aspnetcore.hosting.webhostbuilder) with the `UseConfiguration` method:

[!code-csharp[Main](configuration/sample_snapshot/CommandLine/Program2.cs?highlight=11,15,19)]

---

### Arguments

Arguments passed on the command line must conform to one of two formats shown in the following table.

| Argument format                                                     | Example        |
| ------------------------------------------------------------------- | :------------: |
| Single argument: a key-value pair separated by an equals sign (`=`) | `key1=value`   |
| Sequence of two arguments: a key-value pair separated by a space    | `/key1 value1` |

**Single argument**

The value must follow an equals sign (`=`). The value can be null (for example, `mykey=`).

The key may have a prefix.

| Key prefix               | Example         |
| ------------------------ | :-------------: |
| No prefix                | `key1=value1`   |
| Single dash (`-`)&#8224; | `-key2=value2`  |
| Two dashes (`--`)        | `--key3=value3` |
| Forward slash (`/`)      | `/key4=value4`  |

&#8224;A key with a single dash prefix (`-`) must be provided in [switch mappings](#switch-mappings), described below.

Example command:

```console
dotnet run key1=value1 -key2=value2 --key3=value3 /key4=value4
```

Note: If `-key1` isn't present in the [switch mappings](#switch-mappings) given to the configuration provider, a `FormatException` is thrown.

**Sequence of two arguments**

The value can't be null and must follow the key separated by a space.

The key must have a prefix.

| Key prefix               | Example         |
| ------------------------ | :-------------: |
| Single dash (`-`)&#8224; | `-key1 value1`  |
| Two dashes (`--`)        | `--key2 value2` |
| Forward slash (`/`)      | `/key3 value3`  |

&#8224;A key with a single dash prefix (`-`) must be provided in [switch mappings](#switch-mappings), described below.

Example command:

```console
dotnet run -key1 value1 --key2 value2 /key3 value3
```

Note: If `-key1` isn't present in the [switch mappings](#switch-mappings) given to the configuration provider, a `FormatException` is thrown.

### Duplicate keys

If duplicate keys are provided, the last key-value pair is used.

### Switch mappings

When manually building configuration with `ConfigurationBuilder`, you can optionally provide a switch mappings dictionary to the `AddCommandLine` method. Switch mappings allow you to provide key name replacement logic.

When the switch mappings dictionary is used, the dictionary is checked for a key that matches the key provided by a command-line argument. If the command-line key is found in the dictionary, the dictionary value (the key replacement) is passed back to set the configuration. A switch mapping is required for any command-line key prefixed with a single dash (`-`).

Switch mappings dictionary key rules:

* Switches must start with a dash (`-`) or double-dash (`--`).
* The switch mappings dictionary must not contain duplicate keys.

In the following example, the `GetSwitchMappings` method allows your command-line arguments to use a single dash (`-`) key prefix and avoid leading subkey prefixes.

[!code-csharp[Main](configuration/sample/CommandLine/Program.cs?highlight=10-19,32)]

Without providing command-line arguments, the dictionary provided to `AddInMemoryCollection` sets the configuration values. Run the app with the following command:

```console
dotnet run
```

The console window displays:

```console
MachineName: RickPC
Left: 1980
```

Use the following to pass in configuration settings:

```console
dotnet run /Profile:MachineName=DahliaPC /App:MainWindow:Left=1984
```

The console window displays:

```console
MachineName: DahliaPC
Left: 1984
```

After the switch mappings dictionary is created, it contains the data shown in the following table.

| Key            | Value                 |
| -------------- | --------------------- |
| `-MachineName` | `Profile:MachineName` |
| `-Left`        | `App:MainWindow:Left` |

To demonstrate key switching using the dictionary, run the following command:

```console
dotnet run -MachineName=ChadPC -Left=1988
```

The command-line keys are swapped. The console window displays the configuration values for `Profile:MachineName` and `App:MainWindow:Left`:

```console
MachineName: ChadPC
Left: 1988
```

## The web.config file

A *web.config* file is required when you host the app in IIS or IIS-Express. *web.config* turns on the AspNetCoreModule in IIS to launch your app. Settings in *web.config* enable the AspNetCoreModule in IIS to launch your app and configure other IIS settings and modules. If you are using Visual Studio and delete *web.config*, Visual Studio will create a new one.

## Additional notes

* Dependency Injection (DI) is not set up until after `ConfigureServices` is invoked.
* The configuration system is not DI aware.
* `IConfiguration` has two specializations:
  * `IConfigurationRoot`  Used for the root node. Can trigger a reload.
  * `IConfigurationSection`  Represents a section of configuration values. The `GetSection` and `GetChildren` methods return an `IConfigurationSection`.

## Additional resources

* [Working with Multiple Environments](environments.md)
* [Safe storage of app secrets during development](../security/app-secrets.md)
* [Hosting in ASP.NET Core](xref:fundamentals/hosting)
* [Dependency Injection](dependency-injection.md)
* [Azure Key Vault configuration provider](xref:security/key-vault-configuration)
