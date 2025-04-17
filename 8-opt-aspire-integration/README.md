# Integrating .NET Aspire with existing microservice applications

In the [previous scenario](../8-aspire/), you've already used [.NET Aspire](https://aka.ms/dotnet-aspire) for container orchestration. Let's dig further. How to easily orchestrate your existing microservice apps with .NET Aspire?

## Prerequisites

We will be using the app deployed in the [4-microservices](../4-microservices/).

### Getting the repository root

Initialize the variable `REPOSITORY_ROOT` in your preferred terminal.

```bash
# Bazh/Zsh
REPOSITORY_ROOT=$(git rev-parse --show-toplevel)
```

```powershell
# PowerShell
$REPOSITORY_ROOT = git rev-parse --show-toplevel
```

## Adding .NET Aspire to the Solution

To orchestrate all the apps without Dockerfiles or Docker Compose files, let's add [.NET Aspire](https://aka.ms/dotnet-aspire) to the solution. .NET Aspire is an orchestration tool to easily build and deploy cloud-native applications.

1. Make sure that you're in the `8-opt-aspire-integration/sample` directory.

    ```bash
    cd $REPOSITORY_ROOT/8-opt-aspire-integration/sample
    ```

1. Add .NET Aspire AppHost to the solution.

    ```bash
    dotnet new aspire-apphost -n eShopLite.AppHost -o src/eShopLite.AppHost
    ```

1. Add .NET Aspire ServiceDefault to the solution.

    ```bash
    dotnet new aspire-servicedefaults -n eShopLite.ServiceDefaults -o src/eShopLite.ServiceDefaults
    ```

1. Add both projects to the solution.

    ```bash
    dotnet sln add ./src/eShopLite.AppHost
    dotnet sln add ./src/eShopLite.ServiceDefaults
    ```

1. Add the following references to the `eShopLite.AppHost` project.

    ```bash
    dotnet add ./src/eShopLite.AppHost reference ./src/eShopLite.Products
    dotnet add ./src/eShopLite.AppHost reference ./src/eShopLite.StoreInfo
    dotnet add ./src/eShopLite.AppHost reference ./src/eShopLite.Store
    ```

1. Add the `eShopLite.ServiceDefaults` project to each app as a reference.

    ```bash
    dotnet add ./src/eShopLite.Products reference ./src/eShopLite.ServiceDefaults
    dotnet add ./src/eShopLite.StoreInfo reference ./src/eShopLite.ServiceDefaults
    dotnet add ./src/eShopLite.Store reference ./src/eShopLite.ServiceDefaults
    ```

1. Open `src/eShopLite.AppHost/Program.cs` and add the following codes between `var builder = DistributedApplication.CreateBuilder(args);` and `builder.Build().Run();`.

    ```csharp
    var builder = DistributedApplication.CreateBuilder(args);
    
    // 👇👇👇 Add the codes below.
    
    // Add the Products API app
    var products = builder.AddProject<Projects.eShopLite_Products>("products");
    
    // Add the StoreInfo API app
    var storeinfo = builder.AddProject<Projects.eShopLite_StoreInfo>("storeinfo");
    
    // Add the Store app
    var store = builder.AddProject<Projects.eShopLite_Store>("store")
                       .WithExternalHttpEndpoints()
                       .WithReference(products)
                       .WithReference(storeinfo)
                       .WaitFor(products)
                       .WaitFor(storeinfo);
    
    // 👆👆👆 Add the codes above.
    
    builder.Build().Run();
    ```

1. Open `src/eShopLite.Products/Program.cs` and add the following codes.

    ```csharp
    var builder = WebApplication.CreateBuilder(args);
    
    // 👇👇👇 Add the code below.
    builder.AddServiceDefaults();
    // 👆👆👆 Add the code above.
    
    ...
    
    var app = builder.Build();
    
    // 👇👇👇 Add the code below.
    app.MapDefaultEndpoints();
    // 👆👆👆 Add the code above.
    
    ...
    
    app.Run();
    ```

1. Open `src/eShopLite.StoreInfo/Program.cs` and `src/eShopLite.Store/Program.cs`, and do the same thing as above.

1. Open `src/eShopLite.Store/Program.cs`, find the `builder.Services.AddHttpClient<ProductApiClient>(...` line, and update it with the following code.

    ```csharp
    // Before - 👇👇👇 Remove the lines below
    builder.Services.AddHttpClient<ProductApiClient>(client =>
    {
        var productsApiUrl = builder.Configuration.GetValue<string>("ProductsApi");
        if (string.IsNullOrEmpty(productsApiUrl))
        {
            throw new ArgumentNullException(nameof(productsApiUrl), "ProductsApi configuration value is missing or empty.");
        }
        client.BaseAddress = new Uri(productsApiUrl);
    });
    
    builder.Services.AddHttpClient<StoreInfoApiClient>(client =>
    {
        var storeinfoApiUrl = builder.Configuration.GetValue<string>("StoreInfoApi");
        if (string.IsNullOrEmpty(storeinfoApiUrl))
        {
            throw new ArgumentNullException(nameof(storeinfoApiUrl), "StoreInfoApi configuration value is missing or empty.");
        }
        client.BaseAddress = new Uri(storeinfoApiUrl);
    });
    // Before - 👆👆👆 Remove the lines above
    
    // After - 👇👇👇 Add the lines below
    builder.Services.AddHttpClient<ProductApiClient>(client => client.BaseAddress = new Uri("https+http://products"));
    builder.Services.AddHttpClient<StoreInfoApiClient>(client => client.BaseAddress = new Uri("https+http://storeinfo"));
    // After - 👆👆👆 Add the lines above
    ```

1. Open `src/eShopLite.Store/appsettings.json` and remove the `ProductsApi` and `StoreInfoApi` configurations.

    ```jsonc
    {
      // Remove those two lines
      "ProductsApi": "http://localhost:5228",
      "StoreInfoApi": "http://localhost:5151"
    }
    ```

## Replacing SQLite with PostgreSQL

Let's replace the existing SQLite database with a containerized PostgreSQL one.

1. Make sure that you're in the `8-opt-aspire-integration/sample` directory.

    ```bash
    cd $REPOSITORY_ROOT/8-opt-aspire-integration/sample
    ```

1. Add the PostgreSQL NuGet package to the `eShopLite.AppHost` project.

    ```bash
    dotnet add ./src/eShopLite.AppHost package Aspire.Hosting.PostgreSQL
    ```

1. Add the PostgreSQL NuGet package to both `eShopLite.Products` and `eShopLite.StoreInfo` projects as well.

    ```bash
    dotnet add ./src/eShopLite.Products package Aspire.Npgsql.EntityFrameworkCore.PostgreSQL
    dotnet add ./src/eShopLite.StoreInfo package Aspire.Npgsql.EntityFrameworkCore.PostgreSQL
    ```

   Then, remove the SQLite NuGet package.

    ```bash
    dotnet remove ./src/eShopLite.Products package Microsoft.EntityFrameworkCore.Sqlite
    dotnet remove ./src/eShopLite.StoreInfo package Microsoft.EntityFrameworkCore.Sqlite
    ```

1. Open `src/eShopLite.AppHost/Program.cs`, find the `var builder = DistributedApplication.CreateBuilder(args);` line, and add the following code.

    ```csharp
    var builder = DistributedApplication.CreateBuilder(args);
    
    // 👇👇👇 Add the codes below.
    
    // Add PostgreSQL database
    var pg = builder.AddPostgres("pg")
                    .WithPgAdmin();
    var productsdb = pg.AddDatabase("productsdb");
    var storeinfodb = pg.AddDatabase("storeinfodb");
    
    // 👆👆👆 Add the codes above.
    ```

1. In the same file, find the `var products = builder.AddProject<Projects.eShopLite_Products>("products");` line and update it with the following code.

    ```csharp
    // Before
    var products = builder.AddProject<Projects.eShopLite_Products>("products")
    
    // After
    var products = builder.AddProject<Projects.eShopLite_Products>("products")
                          .WithReference(productsdb)
                          .WaitFor(productsdb);
    ```

1. In the same file, find the `var storeinfo = builder.AddProject<Projects.eShopLite_StoreInfo>("storeinfo");` line and update it with the following code.

    ```csharp
    // Before
    var storeinfo = builder.AddProject<Projects.eShopLite_StoreInfo>("storeinfo");
    
    // After
    var storeinfo = builder.AddProject<Projects.eShopLite_StoreInfo>("storeinfo")
                           .WithReference(storeinfodb)
                           .WaitFor(storeinfodb);
    ```

1. Open `src/eShopLite.Products/Program.cs`, find the `builder.Services.AddDbContext<ProductDbContext>(...` line, and update it with the following code.

    ```csharp
    // Before - 👇👇👇 Remove the lines below
    builder.Services.AddDbContext<ProductDbContext>(options =>
    {
        var connectionString = builder.Configuration.GetConnectionString("ProductsContext") ?? throw new InvalidOperationException("Connection string 'ProductsContext' not found.");
        options.UseSqlite(connectionString);
    });
    // Before - 👆👆👆 Remove the lines above
    
    // After - 👇👇👇 Add the line below
    builder.AddNpgsqlDbContext<ProductDbContext>("productsdb");
    // After - 👆👆👆 Add the line above
    ```

1. Open `src/eShopLite.StoreInfo/Program.cs`, find the `builder.Services.AddDbContext<StoreInfoDbContext>(...` line, and update it with the following code.

    ```csharp
    // Before - 👇👇👇 Remove the lines below
    builder.Services.AddDbContext<StoreInfoDbContext>(options =>
    {
        var connectionString = builder.Configuration.GetConnectionString("StoreInfoContext") ?? throw new InvalidOperationException("Connection string 'StoreInfoContext' not found.");
        options.UseSqlite(connectionString);
    });
    // Before - 👆👆👆 Remove the lines above
    
    // After - 👇👇👇 Add the line below
    builder.AddNpgsqlDbContext<StoreInfoDbContext>("storeinfodb");
    // After - 👆👆👆 Add the line above
    ```

## Running the Microservice Apps with .NET Aspire Locally

1. Make sure that Docker Desktop is running on your machine.

1. Make sure that you're in the `8-opt-aspire-integration/sample` directory.

    ```bash
    cd $REPOSITORY_ROOT/8-opt-aspire-integration/sample
    ```

1. Run the following command to build and run the applications.

    ```bash
    dotnet watch run --project ./src/eShopLite.AppHost
    ```

1. It automatically opens a browser and navigate to `https://localhost:17287` to see the .NET Aspire dashboard is up and running. Please note that the port number might be different from yours.

   ![.NET Aspire Dashboard](./images/8-opt-aspire-integration-01.png)

   As you can see the dashboard, both Products API app and StoreInfo API app now use PostgreSQL instead of SQLite.

1. Click the "View details" menu of the Products API app and see the connection string of the PostgreSQL database.

   ![.NET Aspire Dashboard - PostgreSQL Connection String](./images/8-opt-aspire-integration-02.png)

   And check the StoreInfo API app on the dashboard whether it's pointing to the PostgreSQL database or not.

1. Click the "View details" menu of the Store app and see the connection strings to both Product API and StoreInfo API apps.

   ![.NET Aspire Dashboard - Store App Connection Strings](./images/8-opt-aspire-integration-03.png)

1. Click the Store app link to see the app running. Then navigate to `/stores` and `/products` to see both pages are properly working.

1. To stop the apps, press `Ctrl+C` in a terminal.

## Deploying the Microservice Apps to Azure Container Apps with .NET Aspire via Azure Developer CLI (AZD)

Once you're happy with the .NET Aspire orchestration of all the microservice apps, you can deploy it to ACA through Azure Developer CLI (AZD).

1. Make sure that you're either in the `8-opt-aspire-integration/sample` directory.

    ```bash
    cd $REPOSITORY_ROOT/8-opt-aspire-integration/sample
    ```

1. Initialize the Azure Developer CLI (azd) in the current directory.

    ```bash
    azd init
    ```

   > During initialization, you'll be asked to provide the environment name.

1. Once the initialization is over, you won't be able to see the `infra` directory because it's all managed by .NET Aspire. Instead, open the `azure.yaml` file and see the configurations that only contains the `eShopLite.AppHost` project.

    ```yml
    # yaml-language-server: $schema=https://raw.githubusercontent.com/Azure/azure-dev/main/schemas/v1.0/azure.yaml.json
    
    name: 8-opt-aspire-integration
    services:  
      app:
        language: dotnet
        project: ./src/eShopLite.AppHost/eShopLite.AppHost.csproj
        host: containerapp
    ```

1. Provision and deploy the microservice apps to ACA.

    ```bash
    azd up
    ```

   > While executing this command, you'll be asked to provide the Azure subscription ID and location.

1. Once the deployment is over, you'll see the URLs of the deployed microservice apps on the screen.

   ![Azure Container Apps URLs](./images/8-opt-aspire-integration-04.png)

   Please note that not all the apps are accessible from the public internet because they have `.internal` in the URL while the Store app doesn't have it. The Store app is the only one that has external HTTP endpoints. You will also have the Aspire Dashboard URL.

1. Open your web browser and navigate to the Store app and see the app is up and running on ACA. Then navigate to `/stores` and `/products` to see both pages are properly working.

1. Navigate to the Aspire Dashboard URL to see the status of the deployed apps as well as all the connection strings that are automatically configured by .NET Aspire. Please note that you'll be asked to login to access the dashboard.

## Learn more

Curious of .NET Aspire? Here are a few resources for you to learn more:

- [.NET Aspire](https://aka.ms/dotnet-aspire)
- [Let's Learn .NET - Aspire](https://aka.ms/letslearn/dotnet/aspire)


### Video

[![Getting Started with .NET on ACA - Part 8](../8-aspire/images/ep8_thumb_yt_small.jpg)](https://www.youtube.com/watch?v=Rnn-f-vDS4Y&list=PLI7iePan8aH6jQxDupYUvgQsP3G7WGM0b&index=8)


## Clean up the deployed resources

To clean up the resources, run the following command:

```bash
azd down --force --purge
```

## Up next

🎉 Congratulations! You've completed our **.NET on Azure Container Apps for Beginners** series!
