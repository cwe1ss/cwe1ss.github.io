---
title: Simple zero-downtime updates with ASP.NET Core and health checks
date: 2020-10-22T00:00:00+01:00
categories:
- .NET
---

Do you use a load balancer that isn't tightly integrated with your orchestrator and therefore doesn't know upfront when the orchestrator has to stop an instance of your ASP.NET Core application for an upgrade / a scaling action / a restart?

Does this result in a few failing requests until the load balancer has finally figured out that the instance is gone?

If so, this blog post might be for you!

<!--more-->

## Problem details

We are hosting our ASP.NET Core applications in Azure Service Fabric and public traffic is routed into the cluster via Azure Application Gateway.

Application Gateway doesn't have a direct integration with Service Fabric's naming resolution, so it can't automatically forward traffic to the dynamic ports & nodes of a service in the cluster. Instead, we need to use fixed ports for our ASP.NET Core applications in Service Fabric and we use simple port based routing rules in Application Gateway.

Example: A stateless ASP.NET Core application `fabric:/MyBlog/MyBlogWebsite` is running in our Service Fabric cluster with a fixed port of `5000` and with `InstanceCount=-1` (so it runs on each node). To expose this application, Application Gateway is configured to forward all requests targeting `www.chwe.at` to the fixed `5000`-port on each node in the Service Fabric VMSS (virtual machine scale set).

This works great. However, during application updates, Service Fabric will stop the existing process before it starts the new application version. This is required because the port `5000` has to be released before it can be bound again to the new version. Application Gateway isn't aware of this short termination, so any requests it forwards to the node during that time will fail.

## Health checks to the rescue

Azure Application Gateway (and probably any other load balancer) supports [health probes](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-probe-overview) to decide if it should forward a request to a given node. In the simplest case, it will just periodically do a HTTP request to the root of your application and if it doesn't receive a response or if the response returns a server error, it will take the instance out of rotation after a few failed attempts.

So if one of your application instances gets shut down, the load balancer will stop forwarding traffic to it __after some time__.

*However, this still means that there will be failed requests until that has happened.*

How can we improve this?

*Should we change our deployment process and call an API of our load balancer to actively take the instance out of rotation before we do the update and call another API of the load balancer to take it back in once the new instance is running?* This would definitely work, but unfortunately Azure Application Gateway doesn't have such an API. We would also have to integrate this into every other orchestration action that results in instance shutdowns (scale down, move to another node, ...).

*Wouldn't it be nice if we could just __delay the shutdown__ of our instance and keep serving requests until the load balancer has figured out that it should take the instance out of rotation?*

We can do this in ASP.NET Core by combining the following ideas:
* We need to expose the health status of the application on it's own URL - e.g. `/health`
* With this separate URL, we can switch the health to `Unhealthy`, once the application receives a shutdown signal from the orchestrator (e.g. `CTRL+C`).
* We can now delay the shutdown until the load balancer health-timeout has been reached.
* Until then, we'll just continue to serve any incoming requests.

*You can find the finished code for this post here: [https://github.com/cwe1ss/blog-zero-downtime-with-health-checks](https://github.com/cwe1ss/blog-zero-downtime-with-health-checks). If you want to follow along step by step, look at the separate commits. They area also linked in each step below.*

## Set up the `/health`-endpoint in ASP.NET Core

ASP.NET Core has [a built-in feature for health checks](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-3.1).

To enable it, we need to register the feature with the DI container by calling `services.AddHealhChecks()` in `Startup.ConfigureServices()` and we need to enable the endpoint on the request pipeline by calling `endpoints.MapHealthChecks("/health");` in the `app.UseEndpoints(...)`-block of `Startup.Configure()`.

After this, we can run the app and navigate to `http://localhost:5000/health`. This will return the text "Healthy" and the status code 200.

[See all changes from this step in the Git commit.](https://github.com/cwe1ss/blog-zero-downtime-with-health-checks/commit/0014c70e68ac97e735572be12174d0f9ab3ff7c8)

## Add a health check that switches to Unhealthy, once the application shuts down

We can [add our own health checks](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-3.1#create-health-checks) to the ASP.NET Core health system by implementing the interface `Microsoft.Extensions.Diagnostics.HealthChecks.IHealthCheck`.

To get notified when the application is being shut down, we can use `Microsoft.Extensions.Hosting.IHostApplicationLifetime`. This interface provides a `ApplicationStopping`-hook that is triggered when the shutdown signal is received but before the application stops processing requests!

When combined, we get the following first simple version of our `ShuttingDownHealthCheck`:

```csharp
public class ShuttingDownHealthCheck : IHealthCheck
{
    private HealthStatus _status = HealthStatus.Healthy;

    public ShuttingDownHealthCheck(IHostApplicationLifetime appLifetime)
    {
        appLifetime.ApplicationStopping.Register(() =>
        {
            Console.WriteLine("Shutting down");
            _status = HealthStatus.Unhealthy;
        });
    }

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var result = new HealthCheckResult(_status);
        return Task.FromResult(result);
    }
}
```

Our class also needs to be registered with the DI container by calling `.AddCheck<ShuttingDownHealthCheck>("shutting_down")` on the return object of `services.AddHealthChecks()`.

However, it's important to know that __by default, ASP.NET Core initializes the class for every request__ to the health endpoint. This doesn't work for our scenario as we need the global `_status` variable and just a single `ApplicationStopping`-registration.

To ensure the class is created only once, we have to add it as a singleton to the DI framework via `services.AddSingleton<ShuttingDownHealthCheck>();`.

It's also important to know, that our `ShuttingDownHealthCheck`-class will only be initialized, when it is requested for the first time. So if we just run the app, navigate to http://localhost:5000 and press `CTRL+C`, our "Shutting down" message will NOT appear in the console.

If we navigate to http://localhost:5000/health and press `CTRL+C` afterwards, the "Shutting down" message will appear on the console!

This behavior is fine for our scenario as the load balancer will continuously invoke this endpoint anyway!

[See all changes from this step in the Git commit.](https://github.com/cwe1ss/blog-zero-downtime-with-health-checks/commit/bafd315f72a4555074db2bb4a4e5f15c8c1f0a9b)

## Delay the shutdown

If you've followed the steps so far, you will have noticed that the application still shuts down immediately after "Shutting down" has been printed to the console.

To delay the shutdown, we can simply add a `Thread.Sleep()` to the code in our `ApplicationStopping`-handler. With this, the main thread is blocked but regular requests will still be processed on other threads.

Let's add `Thread.Sleep(TimeSpan.FromSeconds(15));` after our `_status = HealthStatus.Unhealthy;` statement and run the app again.

If we now navigate to http://localhost:5000/health and press `CTRL+C` afterwards, our "Shutting down" message will appear on the console and the application will keep running!

Any request to the `/health`-endpoint during that time will now return "Unhealthy" with a status code 503 (Service unavailable).

When deployed, the load balancer will now receive this `Unhealthy` response and take the instance out of rotation after a few attempts. Until then, any regular requests it sends to the instance will still be processed!

## Improve the health check

There's still a few issues with our custom health check:

* ASP.NET Core has a default shutdown timeout of 5 seconds. After this, it will throw an `OperationCanceledException` and therefore not gracefully shutdown other background services etc.
* The shutdown delay is annoying during development as we now can't quickly close the app.
* Our "Shutting down" message is just printed to the console. It would be nice if it were sent to the regular logging system.

To increase the [ASP.NET Core ShutdownTimeout](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.1#shutdowntimeout-1), we need to configure the `HostOptions` class in `Startup.ConfigureServices()`:

```csharp
services.Configure<HostOptions>(option =>
{
    option.ShutdownTimeout = System.TimeSpan.FromSeconds(30);
});
```

To improve our health check, we'll introduce `IHostEnvironment` to detect if we're running in `Production`-mode and an `ILogger`. Our `ApplicationStopping`-registration will now look like this:

```csharp
appLifetime.ApplicationStopping.Register(() =>
{
    _status = HealthStatus.Unhealthy;

    bool delayShutdown = _hostEnvironment.IsProduction();
    if (delayShutdown)
    {
        var shutdownDelay = TimeSpan.FromSeconds(25);

        _logger.LogInformation(
            "Delaying shutdown for {Seconds} seconds",
            shutdownDelay.TotalSeconds);

        // ASP.NET Core requests are processed on separate threads,
        // so we can just put the main thread on sleep.
        Thread.Sleep(shutdownDelay);

        _logger.LogInformation("Shutdown delay completed");
    }
});
```

Of course, it would also be possible to just skip the registration ouf our health check in the `Startup.ConfigureServices()`-method.

The logic in our ASP.NET Core application is now finished!

[See all changes from this step in the Git commit.](https://github.com/cwe1ss/blog-zero-downtime-with-health-checks/commit/bb5dcecc68d20d5179b8d36fa22f468162e928b8)

## Set the load balancer settings

It's important to understand that we've set a 25 second shutdown delay. This means, the load balancer must take the instance out of rotation before that time. If it fails to do so, there will be failed requests again.

We therefore need to set up our load balancing probes e.g. in the following way:

* __Target URL: /health__
  * *Our custom health endpoint*
* __Interval: 5 seconds__
  * *Run the probe every 5 seconds*
* __Timeout: 4 seconds__
  * *If the service doesn't respond, fail after 4 seconds*
* __Attempts: 3__
  * *Take the service out of rotation after 3 failed attempts*

With these settings, the load balancer will take the service out of rotation after 15-20 seconds!

Of course, you can change these settings to whatever fits your scenario best.

## Summary

This post is quite long as it tries to explain everything step by step, but in general, the idea is very simple:

* We use a custom health check to mark the instance as Unhealthy once the shutdown has been requested
* We delay the shutdown for 25 seconds. Any regular requests will still be processed during that time.
* We make sure the load balancer takes the instance out of rotation before these 25 seconds are over.

You can find the code for this blog here: [https://github.com/cwe1ss/blog-zero-downtime-with-health-checks](https://github.com/cwe1ss/blog-zero-downtime-with-health-checks).

Follow the commits to see the separate steps we've taken.