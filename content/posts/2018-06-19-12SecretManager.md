---
title:  Using the Secret Manager in a Net Core ASP project
author: Will Andrews
date: 2018-06-19
order: 12
---

We've all done it. We have an API key that we accidentally check into our source control, panic and then have to try and remove all traces of it. So I discovered a secret manager that's really simple to use.

Thankfully it is not included as part of the latest .Net Core SDK (2.1 at this time). 

So to use, open up the .csproj file of your .Net Core ASP web application and add in the following to the existing <PropertyGroup> section (There should be the target framework already in there:

```
<PropertyGroup>
    <UserSecretsId>79a3edd0-2092-40a2-a04d-dcb46d5ca9ed</UserSecretsId>
  </PropertyGroup>
```

Note that I've used a GUID inside the tag. This GUID will be used for this project and used as the file reference for this project when the CLI looks for it when prompted.

So now we can start to use it.

Open up the CLI and navigate to your projects folder and run the following command to list all secrets (there won't be any yet but it will prove it's working):

```
dotnet user-secrets list
```
This will show no results, but prove that it's working.

Now to add a secret, use the following:
```
dotnet user-secrets set "Test" "12345"
```
This should give a success message, and then when running the list command again, you should see the secret.

The first string provided (in this instance "Test") is the key name and the second string (in this instance "12345") is the value.

Now to remove, use the following:
```
dotnet user-secrets set "Test"
```
Once again, run the list command and you will see no results.

To remove all secrets, use the following:
```
dotnet user-secrets clear
```

Let's use these secrets in code!!!

In your startup class, in your ConfigureServices method, add in the following to access the secret we added above:
``` c#
string secret = Configuration["Test"];
```

If you have a key in your appsettings.json with the same name as your secret manager, it will automatically use the secret manager version and not your app settings. This means that you don't really need to use appsettings for storing this type of information. Also it means that if you host your app in Azure, you can add your keys and data into the appsettings in the Azure app setting.

And that's it!! Full code below:
``` c#
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    private readonly IConfiguration _configuration;

    // This method gets called by the runtime. Use this method to add services to the container.
    public void ConfigureServices(IServiceCollection services)
    {
        string secret = _configuration["Test"];
        services.AddMvc();
    }

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseMvc();
    }
}
```
