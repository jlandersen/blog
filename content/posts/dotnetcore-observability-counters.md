---
title: "Observing .NET Core Counters (in CloudWatch)"
date: 2020-05-16T10:15:36+02:00
draft: false
tags: [".NET Core", "ASP.NET Core", "DevOps", "AWS", "CloudWatch", "Observability"]
---

Diagnostics have received nice improvements over the [past few .NET Core releases](https://devblogs.microsoft.com/dotnet/introducing-diagnostics-improvements-in-net-core-3-0/). Especially 3.0 brought a good set of improvements and a a [set of tools](https://github.com/dotnet/diagnostics) to diagnose running applications.  While some of it is still in the early days, the cross platform story on runtime diagnostics is improving. A set of metrics I find particularly interesting to continuously optimize is the set of memory / GC related metrics. Especially for high throughput applications, this is particularly interesting, when it might be difficult to replicate production traffic in a test harness.

In this post I will describe extracting from the new [EventCounters](https://github.com/dotnet/diagnostics/blob/master/documentation/design-docs/eventcounters.md) API and ship the counter values to your monitoring system of choice (Prometheus, Datadog, etc.). In this post I will be using CloudWatch.

## Counters in .NET Core
Before .NET Core, you would typically monitor your application with the `PerformanceCounters`. Chances are you poked around with these in Windows Performance Monitor as well. [EventCounters](https://github.com/dotnet/diagnostics/blob/master/documentation/design-docs/eventcounters.md) is the cross platform equivalent of `PerformanceCounters`, which is only supported on Windows. As with Performance Counters, the .NET Core runtime provides a set of default counters out of the box.

[dotnet-counters](https://github.com/dotnet/diagnostics/blob/master/documentation/dotnet-counters-instructions.md) is a global tool available which connects to a running process and observes a set of counters. I recommend you try it out on an application of your own.

As an example, this looks at counters from `System.Runtime` from process 45881 on my computer:
```
dotnet-counters monitor --process-id 45881 System.Runtime

Press p to pause, r to resume, q to quit.
[System.Runtime]
    # of Assemblies Loaded                              14
    % Time in GC (since last GC)                         0
    Allocation Rate (Bytes / sec)               96,118,704
    CPU Usage (%)                                       13
    Exceptions / sec                                     0
    GC Heap Size (MB)                                4,588
    Gen 0 GC / sec                                   1,020
    Gen 0 Size (B)                                      24
    Gen 1 GC / sec                                     420
    Gen 1 Size (B)                              12,583,008
    Gen 2 GC / sec                                      60
    Gen 2 Size (B)                           4,049,819,720
    LOH Size (B)                               537,310,888
    Monitor Lock Contention Count / sec                  1
    Number of Active Timers                              1
    ThreadPool Completed Work Items / sec                2
    ThreadPool Queue Length                              0
    ThreadPool Threads Count                             3
    Working Set (MB)                                 4,664
```

In addition to `monitor`, `dotnet-counters` also supports collecting to a CSV file - however, both are working in an interactive mode. What we want is to have these counters available in our monitoring solution (in this case, CloudWatch).

## EventListener
For in-process listening of events, the [EventListener](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.tracing.eventlistener?view=netcore-3.1) type is available.
In short, this type gives us a one time chance to register as a listener for event sources of the current application domain. I recommend you read through the previously linked documentation on GitHub as the type does certain things that might surprise you (it did to me).

Taken pretty much straight out of the docs, the smallest possible listener you can imagine looks like this:

```csharp
public class SimpleMetricsListener : EventListener
{
    protected override void OnEventSourceCreated(EventSource eventSource)
    {
        if (eventSource.Name.Equals("System.Runtime"))
        {
            EnableEvents(eventSource, EventLevel.LogAlways, EventKeywords.All, new Dictionary<string, string>
            {
                {"EventCounterIntervalSec", "1"}
            });
        }
    }

    protected override void OnEventWritten(EventWrittenEventArgs eventData)
    {
        // Do something with the event
    }
}
```

The two method overrides gives the ability to register for events from a registered event source and reacting to the events. 
When you enable events from a source, you have some basic filtering available, but, keep in mind, you will get everything from that source - also events that are not counters.

With this structure in mind, let's add in the necessary pieces to ship the counters to CloudWatch (the only dependency added here is `AWSSDK.CloudWatch`):

```csharp
public class CloudWatchMetricsListener : EventListener
{
    private readonly EventLevel _level = EventLevel.Verbose;
    private readonly HashSet<string> _sources = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
    {
        "System.Runtime"
    };
    
    private readonly Dictionary<string, StandardUnit> _metricNameToUnit = new Dictionary<string, StandardUnit>()
    {
        { "loh-size", StandardUnit.Bytes },
        { "gen-0-size", StandardUnit.Bytes },
        { "gen-1-size", StandardUnit.Bytes },
        { "gen-2-size", StandardUnit.Bytes },
        { "gen-0-gc-count", StandardUnit.CountSecond },
        { "gen-1-gc-count", StandardUnit.CountSecond },
        { "gen-2-gc-count", StandardUnit.CountSecond },
        { "alloc-rate", StandardUnit.BytesSecond },
    };
    
    private static List<Dimension> _dimensions = new List<Dimension>
    {
        new Dimension() { Name = "InstanceName", Value = Environment.MachineName },
        new Dimension() { Name = "App", Value = "EventCountersDemoApp" },
    };
    
    private readonly IAmazonCloudWatch _cloudWatch;

    public MetricsCollectionListener(IAmazonCloudWatch cloudWatch)
    {
        _cloudWatch = cloudWatch;
    }

    protected override void OnEventSourceCreated(EventSource eventSource)
    {
        if (_sources.Contains(eventSource.Name))
        {
            // Enable events, omitted
        }
    }

    protected override void OnEventWritten(EventWrittenEventArgs eventData)
    {
        if (eventData.EventName != "EventCounters" || eventData.Payload!.Count <= 0 || _cloudWatch is null)
        {
            return;
        }

        var metricData = new List<MetricDatum>(eventData.Payload.Count);
        foreach (var payload in eventData.Payload)
        {
            if (!(payload is IDictionary<string, object> eventPayload))
            {
                continue;
            }

            var metricValues = GetMetricValues(eventPayload);

            if (!_metricNameToUnit.TryGetValue(metricValues.Name, out var unit))
            {
                continue;
            }
            
            metricData.Add(new MetricDatum()
            {
                Value = metricValues.Value,
                MetricName = metricValues.Name,
                Dimensions = _dimensions,
                Unit = unit,
            });
        }

        this._cloudWatch.PutMetricDataAsync(new PutMetricDataRequest()
        {
            Namespace = "Applications",
            MetricData = metricData,
        });
    }
    
    private (string Name, float Value, string MetricType) GetMetricValues(IDictionary<string, object> eventPayload)
    {
        if (!eventPayload.TryGetValue("Name", out var counterName) 
            || !eventPayload.TryGetValue("CounterType", out var counterType))
        {
            return (string.Empty, 0.0f, string.Empty);
        }

        var value = 0.0f;

        if (counterType.Equals("Sum") && eventPayload.TryGetValue("Increment", out var increment))
        {
            value = Convert.ToSingle(increment);
        } 
        else if (counterType.Equals("Mean") && eventPayload.TryGetValue("Mean", out var mean))
        {
            value = Convert.ToSingle(mean);
        }
        
        return (counterName.ToString(), value, counterType.ToString());
    }
}
```

This is all it takes to forward a set of event counters to CloudWatch, or any other monitoring solution.

Some things to consider:
 - Depending on your monitoring solution you may have to massage the values or at least consider how they map to different counter types.
 In this example I keep a mapping from counters to `StandardUnit` in CloudWatch (also acts as a whitelist of which counters to ship).
 
 - We can generalize this type even more and inject configurations with the set of sources and specific counters we want to ship. 
 Be aware that when the `EventListener` constructor runs, it may trigger the overridden methods before your constructor gets a chance to run. This is why I added a null guard for `_cloudWatch` in `OnEventWritten`, even though it is assigned in the constructor. 

Finally, here are the metrics in CloudWatch:

![](/images/2020/05/dotnet-counters-cloudwatch.png)

As a final remark, my guess is that in the future we will see integrations built into the collectors/agents of various monitoring solutions that will do most of this for you. The IPC protocol for extracting events out-of-process (which `dotnet-counters` use) is still under development at the time of writing. Until this is stabilized, custom in-process listeners should do the trick.

