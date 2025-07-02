---
publishDate: 2025-03-30T08:56:16Z
draft: false
title: Opentelemetry .NET SDK Troubleshooting
excerpt: "Discover how to troubleshoot common issues with the OpenTelemetry SDK in .NET applications, focusing on enabling hidden self-diagnostics to quickly identify and resolve configuration errors like endpoint typos."
category: Guide
author: Marcin Brzozowski
image: ../assets/opentelemetry-dotnet-sdk-troubleshooting.hero.png
tags:
  - dotnet
  - opentelemetry
  - back-end
metadata:
  canonical: https://marcinbrzozowski.dev/opentelemetry-dotnet-sdk-troubleshooting
---

Hello, in this short blog post I'd like to go over the challenges related to troubleshooting the OpenTelemetry SDK in .NET applications.

## Problem Statement

Let's start with instrumenting our application with the common OTEL setup:

```csharp
builder.Services
  .AddOpenTelemetry()
  .WithMetrics(metricsBuilder =>
  {
      metricsBuilder.SetResourceBuilder(resourceBuilder);
      
      // Add .NET Runtime instrumentation
      metricsBuilder.AddMeter("System.Runtime");

      // Use HTTP/Protobuf OTLP exporter to push directly to Prometheus 
      metricsBuilder.AddOtlpExporter((exporterOptions, metricReaderOptions) =>
      {
          exporterOptions.Endpoint = new Uri("http://loaclhost:9090/api/v1/otlp/v1/metrics");
          exporterOptions.Protocol = OtlpExportProtocol.HttpProtobuf;
          metricReaderOptions.PeriodicExportingMetricReaderOptions.ExportIntervalMilliseconds = 1000;
      });
```

And then start the application, and you would see similar log messages in the console:

```text
dbug: Microsoft.Extensions.Hosting.Internal.Host[1]
      Hosting starting
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:5001
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
dbug: Microsoft.Extensions.Hosting.Internal.Host[2]
      Hosting started
```

At this point you're probably sure that the .NET Runtime metrics (as configured) will get exported to the telemetry backend (Prometheus in this case), right?
Double-check the Prometheus UI and you will see that there are no metrics there! Why is that? But I've configured everything correctly, or haven't I?

Please take another look at the endpoint configured above:

`exporterOptions.Endpoint = new Uri("http://loaclhost:9090/api/v1/otlp/v1/metrics");` 

Have you notice the typo in the URL? It's a silly mistake, but it can cost you minutes if not hours of debugging and double-checking your telemetry backend configuration (trust me, I've been there, done that 😂).

So why wasn't this logged anywhere as some kind of error, obviously there's no `loaclhost` machine available that is hosting your backend. That would have saved us a lot of trouble.

## Solution

It turns out the there's a way to get informed about the errors happening in the .NET Opentelemetry SDK, which is not that obvious to find.

There's a general guidelines hinting existence of such feature in the docs - a feature named [Self-Diagnostics](https://opentelemetry.io/docs/specs/otel/error-handling/#self-diagnostics):

```quote
All OpenTelemetry libraries – the API, SDK, exporters, instrumentations, etc. – are encouraged to expose self-troubleshooting metrics, spans, and other telemetry that can be easily enabled and filtered out by default.

(...)

Whenever the library suppresses an error that would otherwise have been exposed to the user, the library SHOULD log the error using language-specific conventions. SDKs MAY expose callbacks to allow end users to handle self-diagnostics separately from application code.
```

That doesn't help much in our case, but if you spend some time digging on GitHub you may eventually find this hidden treasure: [OpenTelemetry Readme / Troubleshooting](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry/README.md#troubleshooting).

### Enabling OTEL_DIAGNOSTICS

The [OpenTelemetry Readme / Troubleshooting](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry/README.md#troubleshooting) document describes a self-diagnostic mechanism of the .NET OpenTelemetry SDK that can be enabled by creating a `OTEL_DIAGNOSTICS.json` file in the current working directory of your .NET application, with the following content:

```json
{
    "LogDirectory": ".",
    "FileSize": 32768,
    "LogLevel": "Warning"
}
```

This self-diagnostics mechanism will be logging events from within the OTEL SDK to the log file, located at the location specified by the `LogDirectory` config key. Moreover, you can specify the max log file size and the minimal log level of the events that should be included in the log.

Long story short, simply create a file named `OTEL_DIAGNOSTICS.json` in the CWD of your application (if you are running from IDE like Rider or VS, put this file next to `Program.cs` file) and place the above content inside.

Once you do that, you will start seeing log files created in your filesystem. The logs will be named like so: `OtelDemo.exe.19744.log`. The naming convention is based on the file name of your running application and the process ID. You can see how the name is built by looking at the source code: [SelfDiagnosticsConfigRefresher.OpenLogFile](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry/Internal/SelfDiagnosticsConfigRefresher.cs#L196).

For more details about the implementation of the self-diagnostics logger you can look at its source code [here](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry/Internal/SelfDiagnosticsEventListener.cs) and the related files in the `OpenTelemetry.Internal` namespace.

## Troubleshoot our typo

Now that we know how to enable and use the OTEL_DIAGNOSTICS logger, let's take a look at the content it produced:

```log
If you are seeing this message, it means that the OpenTelemetry SDK has successfully created the log file used to write self-diagnostic logs. This file will be appended with logs as they appear. If you do not see any logs following this line, it means no logs of the configured LogLevel is occurring. You may change the LogLevel to show lower log levels, so that logs of lower severities will be shown.
2025-04-08T19:27:11.2687635Z:LoggerProviderSdk event: '{0}'{Building LoggerProvider.}
2025-04-08T19:27:11.2885719Z:LoggerProviderSdk event: '{0}'{LoggerProviderSdk built successfully.}
2025-04-08T19:27:11.4625705Z:MeterProviderSdk event: '{0}'{Building MeterProvider.}
2025-04-08T19:27:11.5475212Z:MeterProviderSdk event: '{0}'{MeterProvider configuration: {MetricLimit=1000, CardinalityLimit=2000, ExemplarFilter=, ExemplarFilterForHistograms=}.}
```

We get some high-level events from initializing the SDK, then reading further we can find:

```log
2025-04-08T19:27:17.2191205Z:Exporter failed send data to collector to {0} endpoint. Data will not be sent. Exception: {1}{http://loaclhost:9090/api/v1/otlp/v1/metrics}{System.Net.Http.HttpRequestException: No connection could be made because the target machine actively refused it. (loaclhost:9090)
 ---> System.Net.Sockets.SocketException (10061): No connection could be made because the target machine actively refused it.
```

... and BINGO, we have our error! Now the only thing we need to do is to fix the typo, and we're golden 😁

### Summary 

Okay, so in this post we've gone over what self-diagnostics capabilities offered by the OTEL .NET SDK and that is the  OTEL_DIAGNOSTICS logger. Next we've gone through the exercise of enabling & configuring the logger by creating a `OTEL_DIAGNOSTICS.json` file. Lastly we've analyzed the created log file and found our error.

I must admit that this mechanism is far from ideal, as it requires you to manually create some diagnostic file to even use it. Even if you do that, then pulling the logs is not that straightforward, especially when you're running in a Production environment, especially when having ephemeral deployments, e.g. using containers (like Kubernetes), where you'd lose your log file as soon as the pod hosting your application gets removed, which may happen quite often 😊

This is why in the next blog post I will further explore this topic and show you how can you shove those log events into your normal logging pipeline 😍

Cheers!

## Resources

- https://opentelemetry.io/docs/specs/otel/error-handling/#self-diagnostics
- https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry/README.md#troubleshooting
- https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry/Internal/SelfDiagnosticsEventListener.cs
