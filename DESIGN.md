#Introduction
Correlation is .NET library to propagate correlation context across web applications. It handles incoming requests to extract context, intercepts outgoing requests and injects context to them.
Application instrumentation with the library involves several-lines of code.
#Concepts
##Events correlation
![Context](https://cloud.githubusercontent.com/assets/2347409/20274491/a8f0c286-aa49-11e6-8431-d2e9f7cdfbb1.PNG)
Library supports two identifiers:
>-  `correlation-id` identifies operation (transaction, workflow), which may involve multiple services interaction. Stored in HTTP request header `x-ms-request-id`; if header is not present, new Guid is generated.
>- `request-id` identifies particular request; it's **generated** on caller side for an outgoing request and it's scope is limited to this call only.  It allows to distinguish multiple calls from service-a to service-b within the same operation and trace outgoing request on both sides.
By default HTTP request header `x-ms-request-root-id` is used; if header is not present, new Guid is generated. 

Header names are chosen to be compatible with ApplicationInsights SDK. 

##Context
Library provides `CorrelationContext` which is a property bag. `CorrelationId` and `RequestId` are filled by the library, users can add their own properties

`ContextResolver` class provides access to `CorrelationContext` and stores it in `AsyncLocal` varaible

###Context Factory
Library provides `IContextFactory` interface and implementations to parse context from the framework-specific HTTP requests:
- `Microsoft.AspNetCore.Http.HttpRequest` on ASP.NET Core
- `System.Web.HttpRequest` on ASP.NET
- OWIN environment dictionary for OWIN middleware

Applications can, optionally, provide their own implementations of `IContextFactory`
###Context Injector
Library provides `IContextInjector` interface and it's implementations to inject context into the HTTP request (`HttpRequestMessage` or `WebRequest`)
Applications can, optionally, provide their own implementations of `IContextInjector`

###Logging
The library is intended to work with any logging framework. `CorrelationContext` is public and could be retrieved from the `AsyncLocal`, and users are responsible to configure their logging framework to write context along with trace event.

##Request flow
![Request flow](https://cloud.githubusercontent.com/assets/2347409/20283940/f7e77ff8-aa6f-11e6-83cb-454c90307c5c.PNG)

###Incoming requests
Library provides set of framework-specific handlers for incoming request handling:
 >- ASP.NET Core DiagnosticSource listener
`Microsoft.Diagnostics.Correlation.AspNetCore.ContextTracingInstrumentation`
 >- OWIN Middleware
 `Microsoft.Diagnostics.Correlation.Common.Owin.CorrelationContextTracingMiddleware`
 >- HttpModule
 `Microsoft.Diagnostics.Correlation.Http.CorrelationTracingHttpModule`
 >- MVC and WebAPI ActionFilters
 `Microsoft.Diagnostics.Correlation.Mvc.CorrelationTracingFilter`
 `Microsoft.Diagnostics.Correlation.WebApi.CorrelationTracingFilter`

When incoming request is received, library calls `IContextFactory` and stores context in the `AsyncLocal` variable.

[Samples](#samples) section describes how handlers should be used with different frameworks.


###Outgoing requests
Library provides couple of ways to intercept outgoing requests.
When outgoing request is intercepted, it gets context from the `AsyncLocal` and injects context into the request.
Following handlers provided by the library:
>- **DelegatingHandler**
`Microsoft.Diagnostics.Correlation.Common.Http.CorrelationContextRequestHandler` could be added to existing HttpClient pipeline. Developers also could use builders provided by the library to construct `HttpClient`, add other handlers and change inner handler parameters.
>- **DiagnosticListener**
`HttpClient` is instrumented  in .NET Core with [DiagnosticSource](https://docs.microsoft.com/en-us/dotnet/core/api/system.diagnostics.diagnosticsource). It allows to intercept HttpClient calls and instrument outgoing requests.
>- **Profiler** (not shipped)
Library provides capability to instrument `WebRequest.BeginGetResponse` (which would be called for `HttpClient` and `WebRequest` methods).

##Configuration and setup
![Setup](https://cloud.githubusercontent.com/assets/2347409/20234729/5298a40a-a835-11e6-8f22-53e922455433.PNG)
###Customization
For customers who want to extend library functionality or alter it's behavior, we provide way to specify context factory and injectors.

##Nuget packages
>-  **Microsoft.Diagnostics.Correlation.AspNetCore**
Supported on NET 4.X. and .NET Core. This is standalone package which should be used for ASP.NET Core apps.
>- **Microsoft.Diagnostics.Correlation.AspNet**
Supported on	.NET 4.X. Provides ActionFilters for MVC and WebApi on top of Correlation package
>-  **Microsoft.Diagnostics.Correlation**
Supported on	.NET 4.X
This package alone should be used for non ASP.NET apps or ASP.NET apps hosted with OWIN self-host.
>-  **Microsoft.Diagnostics.Correlation.Common**
Supported on .NET 4.X and Core
Contains base interfaces and shared code
>-  **Microsoft.Diagnostics.Context**
Supported on .NET 4.X and Core
Contains `CorrelationContext` and  `ContextResolver` implementations

## Samples
Correlation instrumentation in general consist of 2 steps: 
>- setting up *incoming* request handler. Library provides multiple ways to do it.
>- setting up *outgoing* request handler

### ASP.NET Core
Library provides two instrumentation options:
1. Instrument both, incoming and outgoing requests with `DiagnosticSource`:
`app.UseCorrelationInstrumentation(Configuration.GetSection("Correlation"))`
2. Use `DiagnosticSource` for incoming requests and `DelegatingHandler` for outgoing requests:
  * Incoming requests: call `app.UseCorrelationInstrumentation` with `InstrumentOutgoingRequests` set to `false` in configuration
  * Use `DelegatingHandler` in `HttpClient` pipeline (you may add it as singleton to ASP.NET Core services): `CorrelationHttpClientBuilder.CreateClient()`

**Note**
You can define correlation configuration in the file and call `app.UseCorrelationInstrumentation` with `IConfiguration` parameter.
Sample configuration:
```
"Correlation" : {
    "InstrumentOutgoingRequests" : true,
    "Headers" : {
      "CorrelationIdHeaderName" : "x-custom-correlation-id",
      "RequestIdHeaderName" : "x-custom-request-id",
    },
    "EndpointFilter" : {
        "Allow" : false,
        "Endpoints" : ["core\.windows\.net", "dc\.services\.visualstudio\.com"]
    }
}
```

### ASP.NET
1. Incoming requests handling
In IIS-hosted apps, ActionFilter *must* be used, HttpModule or Owin Middleware *may* be used if context is needed in other custom middlewares or HTTP modules. 
In OWIN self-hosted apps, OWIN middleware *or* filters should be used.
  * **Use WebAPI and/or MVC ActionFilter (required if app is hosted with IIS)**
    * **MVC**: `GlobalFilters.Filters.Add(new Microsoft.Diagnostics.Correlation.Mvc.CorrelationTracingFilter());`
    * **WebAPI**: `GlobalConfiguration.Configuration.Filters.Add(new Microsoft.Diagnostics.Correlation.WebApi.CorrelationTracingFilter());`

  * **Use OWIN middleware (optional)**: `app.Use<CorrelationContextTracingMiddleware>();`
  * **Use HttpModule (optional)**: Add `CorrelationTracingHttpModule` in web.config under `<system.webServer>/<modules>` section (IIS 7.0):
`<add name="CorrelationTracingHttpModule" type="Microsoft.Diagnostics.Correlation.Http.CorrelationTracingHttpModule, Microsoft.Diagnostics.Correlation" preCondition="managedHandler" />`
2. Outgoing requests should be intercepted with instrumented `HttpClient` instance.
Library provide builder to construct instrumented `HttpClient`  and builder allow to customize pipeline and inner handler parameters:
`CorrelationHttpClientBuilder.CreateClient()`.
Consider using dependency injection framework (e.g. [Unity](https://msdn.microsoft.com/en-us/library/dn178463(v=pandp.30).aspx)) to pass `HttpClient` instances to controllers and other components.

### OWIN Self-Hosted apps
1. Incoming requests should be handled with OWIN middleware:
`app.Use<CorrelationContextTracingMiddleware>();`
2. Outgoing requests should be intercepted with instrumented `HttpClient` instance: `CorrelationHttpClientBuilder.CreateClient()`. 
Consider using dependency injection framework (e.g. [Unity](https://msdn.microsoft.com/en-us/library/dn178463(v=pandp.30).aspx)) to pass `HttpClient` instances to controllers and other components.

### ApplicationInsights integration
If you use ApplicationInsights to collect telemetry data, you need to configure  `TelemetryInitializer` to map `CorrelationContext` fields to AppInsights properties:
```
    public class CorrelationTelemetryInitializer : ITelemetryInitializer
    {
        public void Initialize(ITelemetry telemetry)
        {
            var ctx = ContextResolver.GetContext<CorrelationContext>();
            if (ctx != null)
            {
                telemetry.Context.Operation.Id = ctx.CorrelationId;
                telemetry.Context.Operation.ParentId = ctx.RequestId;
	            var operationTelemetry = telemetry as OperationTelemetry;                
                if (operationTelemetry  != null && ctx.ChildRequestId != null)
	                operationTelemetry.Id = ctx.ChildRequestId;
            }
        }
    }
    ....
    //configure CorrelationTelemetryInitializer when application starts
    TelemetryConfiguration.Active.TelemetryInitializers.Add(new CorrelationTelemetryInitializer());
```
#Next Steps
Correlation library is one part of bigger instrumentation topic. It introduces new concepts like the `CorrelationContext` which should be the shared between different instrumentation pieces. 
E.g. If user has ApplicationInsights to collect and send telemetry, this context should be shared by both libraries and potentially user code. Incoming and outgoing requests handling is done by AppInsights and Correlation, so we potentially introduce configuration issues and race conditions when multiple libraries handle incoming requests.
We are considering different options to isolate shared pieces and interfaces.
##Opentracing
[Opentracing](http://opentracing.io/) provides set of interfaces to extract and inject context, start and stop operations (Spans) and describes context.
It, however, neither standartize how services should communicate nor how service should report its telemetry events.
##ILogger
`ILogger` provides interface to work with generic context and log it along with message. However, reporting telemetry may involve different types of events such as traces, requests, metrics.
##ASP.NET (and Core) libraries
Context and interface to access it may become part of the ASP.NET framework
