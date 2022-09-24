# Serilog.Enrichers.Sensitive

[![Build status](https://ci.appveyor.com/api/projects/status/1anco6pj0oovbs54?svg=true)](https://ci.appveyor.com/project/sandermvanvliet/serilog-enrichers-sensitive) [![NuGet Serilog.Enrichers.Sensitive](https://buildstats.info/nuget/Serilog.Enrichers.Sensitive)](https://www.nuget.org/packages/Serilog.Enrichers.Sensitive/)

This is a Serilog enricher that can mask sensitive data from a `LogEvent` message template and its properties. Currently this supports e-mail addresses and IBAN numbers but could easily be extended to other types of data.

There are two ways of using this enricher:

- Always mask sensitive data (default behaviour)
- Mask data in sensitive areas only

See [Usage](#usage) below on how to configure this.

## Possible use case

Let's say you have written a request/response logging middleware for ASP.Net Core that outputs:

`Request start {method} {url}`
`End {method} {status_code} {url} {duration}`

Here you have the potential that the `url` property contains sensitive data because someone might do `GET /api/users/?email=james.bond@universalexports.co.uk`.

Of course you can write your logging middleware to capture this and that may be the best place in this situation. However there might be cases where you don't know this is likely to happen and then you end up with the e-mail address in your logging platform.

When using this enricher what you will get is that the log message that used to be:

`Request start GET /api/users/?email=james.bond@universalexports.co.uk`

will be:

`Request start GET /api/users/?email=***MASKED***`

### It does not end here

Even though that you know the sensitive data will be masked, it is good practice to not log sensitive data *at all*.

The good thing is that with the masking applied you can add an alert to your logging platform that scans for `***MASKED***` and gives you feedback when sensitive data has been detected. That allows you to fix the problem where it originates (the logging middleware).

## Usage

### Always mask sensitive data

Configure your logger with the enricher:

```csharp
var logger = new LoggerConfiguration()
    .Enrich.WithSensitiveDataMasking()
    .WriteTo.Console()
    .CreateLogger();
```

If you then have a log message that contains sensitive data:

```csharp
logger.Information("This is a sensitive {Email}", "james.bond@universalexports.co.uk");
```

the rendered message will be logged as:

`This is a sensitive ***MASKED***`

the structured log event will look like (abbreviated):

```json
{
    "RenderedMessage": "This is a sensitive ***MASKED***",
    "message": "This is a sensitive {Email}",
    "Properties.Email": "***MASKED***"
}
```

### Mask in sensitive areas only

Configure your logger with the enricher:

```csharp
var logger = new LoggerConfiguration()
    .Enrich.WithSensitiveDataMaskingInArea()
    .WriteTo.Console()
    .CreateLogger();
```

in your application you can then define a sensitive area:

```csharp
using(logger.EnterSensitiveArea())
{
    logger.Information("This is a sensitive {Email}", "james.bond@universalexports.co.uk");
}
```

The effect is that the log message will be rendered as:

`This is a sensitive ***MASKED***`

See the [Serilog.Enrichers.Sensitive.Demo](src/Serilog.Enrichers.Sensitive.Demo/Program.cs) app for a code example of the above.

### Configuring masking operators to use

By default the enricher uses the following masking operators:

- EmailAddressMaskingOperator
- IbanMaskingOperator
- CreditCardMaskingOperator

It's good practice to only configure the masking operators that are applicable for your application. For example:

```csharp
new LoggerConfiguration()
    .Enrich
    .WithSensitiveDataMasking(
        options =>
        {
            options.MaskingOperators = new List<IMaskingOperator> 
            {
                new EmailAddressMaskingOperator(),
                new IbanMaskingOperator()
                // etc etc
            };
        });
```csharp

It is also possible to not use any masking operators but instead mask based on property names. In that case you can configure the enricher to not use any masking operators at all:

```csharp
new LoggerConfiguration()
    .Enrich
    .WithSensitiveDataMasking(
        options =>
        {
            options.MaskingOperators.Clear();
        });
```csharp

### Using a custom mask value

In case the default mask value `***MASKED***` is not what you want, you can supply your own mask value:

```csharp
var logger = new LoggerConfiguration()
    .Enrich.WithSensitiveDataMasking(options => options.MaskValue = "**")
    .WriteTo.Console()
    .CreateLogger();
```

A example rendered message would then look like:

`This is a sensitive value: **`

You can specify any mask string as long as it's non-null or an empty string.

### Always mask a property

It may be that you always want to mask the value of a property regardless of whether it matches a pattern for any of the masking operators. In that case you can specify that the property is always masked:

```csharp
var logger = new LoggerConfiguration()
    .Enrich.WithSensitiveDataMasking(options => options.MaskProperties.Add("email"))
    .WriteTo.Console()
    .CreateLogger();
```

> **Note:** The property names are treated case-insensitive. If you specify `EMAIL` and the property name is `eMaIL` it will still be masked.

When you log any message with an `email` property it will be masked:

```csharp
logger.Information("This is a sensitive {Email}", "this doesn't match the regex at all");
```

the rendered log message comes out as: `"This is a sensitive ***MASKED***"`

### Never mask a property

It may be that you never want to mask the value of a property regardless of whether it matches a pattern for any of the masking operators. In that case you can specify that the property is never masked:

```csharp
var logger = new LoggerConfiguration()
    .Enrich.WithSensitiveDataMasking(options => options.ExcludeProperties.Add("email"))
    .WriteTo.Console()
    .CreateLogger();
```

> **Note:** The property names are treated case-insensitive. If you specify `EMAIL` and the property name is `eMaIL` it will still be excluded.

When you log any message with an `email` property it will not be masked:

```csharp
logger.Information("This is a sensitive {Email}", "user@example.com");
```

the rendered log message comes out as: `"This is a sensitive user@example.com"`


## Extending to additional use cases

Depending on the type of masking operation you want to perform, the `RegexMaskingOperator` base class is most likely your best starting point. It provides a number of extension points:

| Method | Purpose |
|--------|---------|
| ShouldMaskInput | Indicate whether the operator should continue with masking the input |
| PreprocessInput | Perform any operations on the input value before masking the input |
| PreprocessMask | Perform any operations on the mask before masking the matched value | 
| ShouldMaskMatch | Indicate whether the operator should continue with masking the matched value from the input | 

To implement your own masking operator, inherit from `RegexMaskingOperator`, supply the regex through the base constructor and where necessary override any of the above extension points.

Then, when configuring your logger, pass your new encricher in the collection of masking operators:

```csharp
var logger = new LoggerConfiguration()
    .Enrich.WithSensitiveDataMasking(options => {
        // Add your masking operator:
        options.MaskingOperators.Add(new YourMaskingOperator());
    })
    .WriteTo.Console()
    .CreateLogger();
```
