# Integrating Scalar API Reference in ASP.NET Web API (.NET 9+)

This guide provides the steps to add the Scalar API reference to your ASP.NET Web API, including necessary configuration for Bearer Authentication in .NET 9 and .NET 10.

## 1\. Installation

First, you need to add the `Scalar.AspNetCore` package to your project.

```bash
# Using the dotnet CLI
dotnet add package Scalar.AspNetCore

# Or via the NuGet Package Manager Console
Install-Package Scalar.AspNetCore
```

## 2\. Basic Scalar Configuration (Program.cs)

In your `Program.cs` file, register the Scalar middleware.

### Required Mapping

Add the following line, typically after `app.MapOpenApi();`:

```csharp
app.MapScalarApiReference();
```

### Optional Configuration with Options

You can customize the appearance and behavior of the Scalar documentation using an options lambda:

```csharp
app.MapScalarApiReference(options =>
{
    options.WithTitle("Api Reference")
        .WithTheme(ScalarTheme.Kepler)
        .WithDefaultHttpClient(ScalarTarget.CSharp, ScalarClient.HttpClient);
});
```

-----

## 3\. Configuring Bearer Authentication (JWT)

To support Bearer (JWT) authentication in the Scalar documentation, you need to configure an `IOpenApiDocumentTransformer` to inject the security scheme into your OpenAPI document.

### Step 3a: The Security Transformer

The implementation of the `BearerSecuritySchemeTransformer` changes slightly between .NET 9 and .NET 10 due to evolving syntax and types.

#### ⚙️ .NET 9 Implementation

Create a class named `BearerSecuritySchemeTransformer` that implements `IOpenApiDocumentTransformer`.

```csharp
internal sealed class BearerSecuritySchemeTransformer : IOpenApiDocumentTransformer
{
    private readonly IAuthenticationSchemeProvider _authenticationSchemeProvider;

    public BearerSecuritySchemeTransformer(IAuthenticationSchemeProvider authenticationSchemeProvider)
    {
        _authenticationSchemeProvider = authenticationSchemeProvider;
    }

    public async Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken cancellationToken)
    {
        var authenticationSchemes = await _authenticationSchemeProvider.GetAllSchemesAsync();

        if (authenticationSchemes.Any(authScheme => authScheme.Name == JwtBearerDefaults.AuthenticationScheme))
        {
            var requirements = new Dictionary<string, OpenApiSecurityScheme>
            {
                [JwtBearerDefaults.AuthenticationScheme] = new OpenApiSecurityScheme
                {
                    Type = SecuritySchemeType.Http,
                    Scheme = "bearer",
                    In = ParameterLocation.Header,
                    BearerFormat = "JWT"
                }
            };

            document.Components ??= new OpenApiComponents();
            document.Components.SecuritySchemes = requirements;

            foreach (var operation in document.Paths.Values.SelectMany(path => path.Operations))
            {
                operation.Value.Security.Add(new OpenApiSecurityRequirement
                {
                    [new OpenApiSecurityScheme
                    {
                        Reference = new OpenApiReference
                        {
                            Id = JwtBearerDefaults.AuthenticationScheme,
                            Type = ReferenceType.SecurityScheme
                        }
                    }] = []
                });
            }
        }
    }
}
```

#### ⚙️ .NET 10 Implementation

The .NET 10 version utilizes primary constructors and the updated `IOpenApiSecurityScheme` type.

```csharp
internal sealed class BearerSecuritySchemeTransformer(IAuthenticationSchemeProvider authenticationSchemeProvider)
    : IOpenApiDocumentTransformer
{
    public async Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context,
        CancellationToken cancellationToken)
    {
        IEnumerable<AuthenticationScheme> authenticationSchemes =
            await authenticationSchemeProvider.GetAllSchemesAsync();

        if (authenticationSchemes.Any(authScheme => authScheme.Name == JwtBearerDefaults.AuthenticationScheme))
        {
            Dictionary<string, IOpenApiSecurityScheme> requirements = new()
            {
                [JwtBearerDefaults.AuthenticationScheme] = new OpenApiSecurityScheme()
                {
                    Type = SecuritySchemeType.Http,
                    Scheme = "bearer",
                    In = ParameterLocation.Header,
                    BearerFormat = "JWT"
                }
            };

            document.Components ??= new OpenApiComponents();
            document.Components.SecuritySchemes = requirements;

            foreach (KeyValuePair<HttpMethod, OpenApiOperation> operation in document.Paths.Values.SelectMany(path =>
                         path.Operations!))
            {
                operation.Value.Security?.Add(new OpenApiSecurityRequirement
                {
                    [new OpenApiSecuritySchemeReference(JwtBearerDefaults.AuthenticationScheme)] = []
                });
            }
        }
    }
}
```

### Step 3b: Register the Transformer (Program.cs)

In `Program.cs`, modify the `builder.Services.AddOpenApi()` call to register your new transformer:

```csharp
// Before:
// builder.Services.AddOpenApi();

// After (Add the transformer):
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer<BearerSecuritySchemeTransformer>();
});
```

### Step 3c: Configure Scalar for Authentication

Finally, update the `app.MapScalarApiReference()` middleware to explicitly prefer the Bearer security scheme.

```csharp
app.MapScalarApiReference(options =>
{
    options.Authentication = new ScalarAuthenticationOptions
    {
        PreferredSecuritySchemes = [JwtBearerDefaults.AuthenticationScheme]
    };
});
```
