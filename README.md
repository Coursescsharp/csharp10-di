# Adding a DI container

Install the **Microsoft.Extensions.Hosting** NuGet package. It contains DI approach for hosting applications which includes the **Microsoft.Extensions.DependencyInjection** package as a dependency.

This provides us with just not a DI container, but also other sensible defaults for, for example, configuration.

Then in the code (Program.cs) we will add the code for creating a default builder for the application host. It can be used to host several things, including DI.

```
using var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {

    })
    .Build();
```

For adding DI, we wil need to configure the builder by calling into another function called **ConfigureServices**. It takes a lambda function as an argument, and the lambda expression takes two parameters: the host
builder context and the second one is the service collection.

This is the one we will be using to register types in the classes we want to be using. So, inside the lambda expression we have to list all the classes we will be using, so that the container can instantiate classes
for us:

```
using var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        services.AddTransient<ProductImporter>();
    })
    .Build();
```

This will make our classes known to the container. Now, to instantiate the ProductImporter classes using the DI container, we use the application host to get an instance. That is done getting the service provider, 
and that collection is built by the application host as part of invoking the Build method above.

```
var productImporter = host.Services.GetRequiredService<ProductImporter>();
productImporter.Run();
```

The container now will look and see if there is a registration for that type, and if there is, it will instantiate it for us. But all the classes used by the ProductImporter hae to be registered as well.

For that, we will add a **Configuration** class:

```
using var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        services.AddTransient<Configuration>();
        services.AddTransient<IPriceParser, PriceParser>();
        services.AddTransient<ProductImporter>();
    })
    .Build();
...

var configuration = new Configuration();
```

In the case of **PriceParser**, instead of registering the type directly, we register both the interface and the class. This is because the ProductImporter class declares a dependency on the  IPriceParser interface, 
but at runtime we want it to work with the implementing type, the class.

Adding a registration this way, the container will return a PriceParser whenever an IPriceParser is requested.

```
using var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        services.AddTransient<Configuration>();
        services.AddTransient<IPriceParser, PriceParser>();
        services.AddTransient<IProductSource, ProductSource>();
        
        services.AddTransient<IProductFormatter, ProductFormatter>();
        services.AddTransient<IProductTarget, ProductTarget>();
        
        services.AddTransient<ProductImporter>();
    })
    .Build();

var productImporter = host.Services.GetRequiredService<ProductImporter>();
productImporter.Run();
```

In this way, through DI:

* No longer calling constructors
* container calls the constructors
* Not concerned with ordering registrations
* Not concerned with dependencies of each type

I only register the types I want to make available.

## About DI Containers

**Well-known DI Containers**

* **Autofac**
* **Ninject**
* **Microsoft.Extensions.DependencyInjection (default for .NET Core and .NET 5 and up)**

### Working with a DI Container

When working with a DI Container, we'll always work with it in one or two phases or modes:

**Registration phase**

We register types in the container, so it knows of their existence and when to construct them. We're doing this so that the container knows about them and can resolve them later.

All containers allow us to provide two types when making a registration. They are called the service type and the implementing type.

Finally, during registration, we can also specify a lifetime. It determines whether the DI container will instantiate a new service type or return an existing instance.

**Resolving phase**

The container is responsible for instantiating types and providing them when requested.

The second resposability of the container in this phase is to manage the lifetime of our classes. It will decide to instantiate a new type or use an existing one and dispose of the types when it is sure we can no
longer request them.