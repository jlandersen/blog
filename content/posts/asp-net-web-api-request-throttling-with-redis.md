+++
author = "Jeppe Lund Andersen"
categories = ["C#", "ASP.NET", "Redis", "Azure", "REST", "Web API"]
date = 2015-08-16T12:26:52Z
slug = "asp-net-web-api-request-throttling-with-redis"
tags = ["C#", "ASP.NET", "Redis", "Azure", "REST", "Web API"]
title = "ASP.NET Web API Request Throttling with Redis Cache"

+++

When developing API's for external parties to consume, sometimes the need for request throttling becomes relevant. In this article you'll see a simple approach to create such a middleware component for Web API in ASP.NET 5, based on Redis.

You can browse or download the complete sample on GitHub at: https://github.com/jlandersen/web-api-redis-request-throttling.

# Considerations
The general case usually is that you have identifiable consumers of your API. The consumers are usually identified based on an access key (or something equivalent) supplied with each request. Since you will have to check each incoming request up against a history of previous requests, the mechanism should almost always be implemented using in-memory caching or a dedicated caching system, such as memcached or Redis.
Before implementing a request throttling mechanism in your API, that rates each of these consumers individually, you should consider how ambitious the mechanism should be. You can choose from a wide range of different parameters to base the rating on. Some examples are:


* Number of requests
* Raw payload size
* Number of serialized objects in the body

Once you have the throttling parameter in order, consider how it should work. This is where you can easily over complicate the situation. You could decide that allowing up to 100 requests per minute should be based on a "rolling" scheme, such that for each consumer you always know the current request per minute rate at all times. The implementation can be done in several ways, but will need to be aware of timing and/or have a background task that updates the consumers request counter every second.

Instead, consider an implementation that allows for 100 requests every minute by conceptually giving each consumer a total of 100 requests to use. This total is then simply replenished every minute. It may result in the consumer having to back off for a little longer, should they reach the limit, but you can alleviate this by tweaking the total allowed requests and interval between a replenishment (e.g. allow 25 requests every 15 seconds).

# Throttling Requests with Redis Cache 
Let us see how the above described scenario can be implemented using Redis. I am going to use a Redis instance hosted on Azure, but you should just have any Redis instance(s) going for this to work. I use the [StackExchange.Redis](https://github.com/StackExchange/StackExchange.Redis) client, available via nuget.

We will use the [HINCRBY](http://redis.io/commands/HINCRBY) command to implement the scenario. The idea is to create a number stored at a key for each consumer which upon creation will have an expiry of one minute. On each request the number is incremented. If the number goes above the threshold, a HTTP 429 response will be returned.

HINCRBY returns the value after the increment operation, and if no value exists it will set it to 0 before the increment operation. We can use this to implement the above scenario as follows:

```csharp
// This should be reused - see section on Middleware
var connection = ConnectionMultiplexer.Connect(
    "[redis instance],ssl=true,password=[access key]");
var cache = connection.GetDatabase();

// Replace this with an extraction of whatever you use to identity consumers of the API
var consumerKey = Guid.NewGuid().ToString();

var consumerThrottleCacheKey = $"consumer.throttle#{consumerKey}";
var cacheResult = cache.HashIncrement(consumerThrottleCacheKey, 1);

if (cacheResult == 1)
{
    cache.KeyExpire(consumerThrottleCacheKey, TimeSpan.FromSeconds(60));
}
else if (cacheResult > requestsPerMinuteThreshold)
{
    // Return a 429 response
}

// Otherwise continue processing the request as normal
```

Note that in this case I maintain a cache entry with a unique cache key for each consumer. Since HINCRBY works with multiple fields stored at the same cache key, you could also let each consumer be an individual field of the same cache key. Since you cannot expire values at the field level, you will need to do a separate check if the key exists instead before performing an increment.

# Use as Middleware
Let us see how to put this into proper use as middleware in ASP.NET 5 supported by the built-in dependency injection system. 

StackExchange recommends reusing the `ConnectionMultiplexer`, so let us take care of this first. In the `ConfigureServices`method in the `Startup` class, register the connection object as a singleton (or scoped, but not transient!):
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
    services.AddSingleton<IConnectionMultiplexer>(i => ConnectionMultiplexer.Connect("connection"));
}
```

Next we create the middleware component. This will take in the threshold as an argument (the other constructor arguments will be supplied by the framework):
```csharp
public class RequestThrottlingMiddleware
{

    private IConnectionMultiplexer connection;
    private RequestDelegate next;
    private int requestsPerMinuteThreshold;

    public RequestThrottlingMiddleware(
          RequestDelegate next, 
          IConnectionMultiplexer connection, 
          int requestsPerMinuteThreshold)
    {
        this.next = next;
        this.connection = connection;
        this.requestsPerMinuteThreshold = requestsPerMinuteThreshold;
    }

    public async Task Invoke(HttpContext context)
    {
        var cache = connection.GetDatabase();

        // Get this from the context in whatever way the user supplies it
        var consumerKey = Guid.NewGuid().ToString();

        var consumerCacheKey = $"consumer.throttle#{consumerKey}";

        var cacheResult = cache.HashIncrement(consumerCacheKey, 1);

        if (cacheResult == 1)
        {
            cache.KeyExpire(
               $"consumer.throttle#{consumerKey}", 
               TimeSpan.FromSeconds(60), 
               CommandFlags.FireAndForget);
        }
        else if (cacheResult > requestsPerMinuteThreshold)
        {
            context.Response.StatusCode = 429;

            using (var writer = new StreamWriter(context.Response.Body))
            {
                await writer.WriteAsync("You are making too many requests.");
            }

            return;
        }

        await next(context);
    }
}
```

We can now use the `RequestThrottlingMiddleware` by registering in the `Configure` method in the `Startup` class:
```csharp
app.UseMiddleware<RequestThrottlingMiddleware>(100);
```