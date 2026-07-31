---
name: implement-api-endpoints
description: Create, modify, migrate, or review HTTP API endpoints in API projects using FastEndpoints. Use for any business route under src/API, endpoint request or response contracts, endpoint dependency injection, authentication or authorization metadata, API integration slices, endpoint HTTP request files, or replacement of ASP.NET Minimal API MapGet, MapPost, MapPut, MapPatch, and MapDelete handlers.
---

# Implement API endpoints

## Inspect the complete contract

Before editing:

1. Read `AGENTS.md` and any feature-specific skill it names.
2. Find the current route, HTTP method, endpoint name, request and response types, status codes, access rules, downstream service or workflow, tests, and `.http` request.
3. Trace the complete feature slice through Application, Infrastructure, API, MCP, configuration, documentation, and deployment files.
4. Preserve established behavior unless the user explicitly requests a contract change.

## Use FastEndpoints

Implement every business API route as a sealed class in `src/API/Endpoints`:

```csharp
public sealed class ExampleEndpoint(
    IExampleWorkflow workflow)
    : Endpoint<ExampleRequest, ExampleResponse>
{
    public override void Configure()
    {
        Post("/api/v1/examples");
        AllowAnonymous();
        Options(builder => builder.WithName("CreateExample"));
    }

    public override async Task HandleAsync(
        ExampleRequest request,
        CancellationToken cancellationToken)
    {
        var response = await workflow.RunAsync(request, cancellationToken);

        await Send.OkAsync(response);
    }
}
```

Apply these rules:

- Derive from the appropriate FastEndpoints base type: `Endpoint<TRequest, TResponse>`, `Endpoint<TRequest>`, `EndpointWithoutRequest<TResponse>`, or `EndpointWithoutRequest`.
- Keep one endpoint as the primary type in each file and align the file name with the type name.
- Use the `API.Endpoints` namespace.
- Configure the HTTP verb, absolute route, access rules, and stable endpoint name in `Configure()`.
- Preserve existing authentication and authorization. Call `AllowAnonymous()` only for a deliberately public endpoint.
- Inject workflows or integration interfaces through the constructor. Do not resolve services manually from `HttpContext`.
- Pass the supplied cancellation token through all asynchronous calls.
- Use typed request and response contracts. Keep business orchestration out of the endpoint.
- Return the established status code with `Send.*Async()` or a typed-result response. Do not silently translate failures or change response envelopes.
- Preserve external product-owned JSON as `JsonElement` when ENGINE does not own the response schema.
- Do not add business routes with `app.MapGet`, `app.MapPost`, `app.MapPut`, `app.MapPatch`, or `app.MapDelete`.
- Keep `AddFastEndpoints()` and `UseFastEndpoints()` registered once in `src/API/Program.cs`. Do not duplicate startup registration for each feature.
- Retain `MapDefaultEndpoints()` for Aspire health and liveness infrastructure; it is not a business API implementation pattern.

## Place responsibilities correctly

- Put transport binding and HTTP metadata in API.
- Put use-case orchestration and validation in Application workflows or services.
- Put external HTTP, persistence, and platform implementations in Infrastructure.
- Put ENGINE-owned business entities in Domain and follow `.github/skills/use-domain-model-base/SKILL.md`.
- Reuse existing Application request types when they already express the endpoint contract.
- Keep API and MCP as separate transports over the same Application behavior. When a feature is exposed through both, update and validate both surfaces together.
- Do not duplicate downstream product models in ENGINE merely to make an endpoint response more specific.

## Add request coverage

Create or update one request file per endpoint under `src/API/tests/http`:

- Name it `<EndpointType>.http`.
- Keep the route, method, headers, and body synchronized with the endpoint contract.
- Reuse `{{API_HostAddress}}` and existing environment-variable conventions.
- Do not create a combined `API.http` file.
- Never place credentials, tokens, or connection strings in request examples.

Add focused automated tests when behavior, validation, status mapping, serialization, or authorization changes. Prefer testing Application behavior independently from transport details, then add endpoint-level coverage for HTTP-specific contracts when the repository has the required test host.

## Validate the result

Before completing endpoint work:

1. Search `src/API` for business route mappings outside FastEndpoints:

   ```powershell
   rg -n "Map(Get|Post|Put|Delete|Patch)|ControllerBase|ApiController" src/API -g "*.cs"
   ```

   Inspect matches rather than removing `MapDefaultEndpoints()`.

2. Confirm every endpoint class inherits a FastEndpoints base type and is discoverable from the API assembly.
3. Confirm route, verb, endpoint name, access rules, request shape, response shape, and status behavior match the intended contract.
4. Confirm each endpoint has its own current `.http` request file.
5. Run:

   ```powershell
   dotnet build src/[YourSolutionName].slnx.
   dotnet test src/[YourSolutionName].slnx --no-build.
   git diff --check
   ```
Remember to replace `[YourSolutionName]` with your actual solution name.
6. Report the endpoints changed, preserved contracts, and exact validation results. A successful build is mandatory.
