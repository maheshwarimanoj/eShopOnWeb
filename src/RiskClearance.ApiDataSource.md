# Risk Clearance — Handling API Data Sources

## The Request Stays the Same

The caller doesn't know or care whether a data source is an internal DB or external API.

```json
{
  "correlationId": "RC-20260317-000456",
  "requestTimestamp": "2026-03-17T12:45:00Z",
  "sourceSystem": "Rx0",
  "sourceSystemReference": "RX0-REQ-0002",

  "insuredDetails": [
    {
      "insuredName": "CocaCola Inc",
      "uniqueId": "123456789",
      "countryCode": "US"
    }
  ],

  "coverageDetails": {
    "classOfBusiness": ["PVT"]
  },

  "searchParameters": {
    "transactionTypes": ["Policy", "Quote"],
    "sourceProviderName": "GSIDataPlatform",
    "dataSources": ["S2000", "Majesco", "LloydsMDR"],
    "confidenceScore": 50,
    "lookBackMonths": 6,
    "numberOfResults": 10,
    "statusFilter": ["Bound", "Cancelled"]
  }
}
```

S2000 and Majesco hit internal databases. LloydsMDR hits an external API.
Same request. Same shape. The **adapter layer** handles the difference.

---

## Updated Enum — Tag Each Source

```csharp
public enum DataSource
{
    // Internal (DB-backed)
    S2000,
    Majesco,
    Rx0,

    // External (API-backed)
    LloydsMDR,
    CedantApi,
    MGAPortal
}
```

---

## Data Source Registry (Config-Driven)

Rather than hardcoding which source is internal vs API, use a registry.
This lives in `appsettings.json` or Azure App Configuration:

```json
{
  "DataSourceRegistry": {
    "S2000": {
      "type": "Internal",
      "connectionString": "Server=...;Database=S2000;..."
    },
    "Majesco": {
      "type": "Internal",
      "connectionString": "Server=...;Database=Majesco;..."
    },
    "Rx0": {
      "type": "Internal",
      "connectionString": "Server=...;Database=Rx0;..."
    },
    "LloydsMDR": {
      "type": "ExternalApi",
      "baseUrl": "https://api.lloyds.com/mdr/v1",
      "authType": "OAuth2_ClientCredentials",
      "tokenEndpoint": "https://auth.lloyds.com/token",
      "timeoutSeconds": 30,
      "retryCount": 3
    },
    "CedantApi": {
      "type": "ExternalApi",
      "baseUrl": "https://api.cedant.com/risks/v2",
      "authType": "ApiKey",
      "timeoutSeconds": 15,
      "retryCount": 2
    }
  }
}
```

---

## Config Model

```csharp
public sealed record DataSourceConfig
{
    public required DataSourceType Type { get; init; }
    public string? ConnectionString { get; init; }    // Internal
    public string? BaseUrl { get; init; }              // ExternalApi
    public AuthType AuthType { get; init; }            // ExternalApi
    public string? TokenEndpoint { get; init; }        // OAuth2
    public int TimeoutSeconds { get; init; } = 30;
    public int RetryCount { get; init; } = 3;
}

public enum DataSourceType
{
    Internal,
    ExternalApi
}

public enum AuthType
{
    None,
    ApiKey,
    OAuth2_ClientCredentials,
    ManagedIdentity
}
```

---

## Adapter Pattern — One Interface, Two Implementations

```csharp
/// <summary>
/// Every data source — internal or API — implements this.
/// The domain layer only knows this interface.
/// </summary>
public interface IRiskDataSourceAdapter
{
    DataSource Source { get; }
    Task<IReadOnlyList<RiskMatch>> SearchAsync(
        RiskSearchContext context,
        CancellationToken ct);
}

/// <summary>
/// The search context passed to every adapter — 
/// built from the common domain model.
/// </summary>
public sealed record RiskSearchContext
{
    public required IReadOnlyList<InsuredDetail> InsuredDetails { get; init; }
    public required IReadOnlyList<TransactionType> TransactionTypes { get; init; }
    public required IReadOnlyList<ClassOfBusiness> ClassOfBusiness { get; init; }
    public required IReadOnlyList<TransactionStatus> StatusFilter { get; init; }
    public required int LookBackMonths { get; init; }
    public required int ConfidenceScore { get; init; }
    public required int NumberOfResults { get; init; }
}

/// <summary>
/// Common response — every adapter returns this same shape.
/// </summary>
public sealed record RiskMatch
{
    public required string MatchId { get; init; }
    public required DataSource Source { get; init; }
    public required TransactionType TransactionType { get; init; }
    public required string InsuredName { get; init; }
    public required decimal ConfidenceScore { get; init; }
    public required TransactionStatus Status { get; init; }
    public required DateTimeOffset EffectiveDate { get; init; }
    public Dictionary<string, object>? AdditionalData { get; init; }
}
```

---

## Internal Adapter (DB)

```csharp
public class S2000Adapter : IRiskDataSourceAdapter
{
    private readonly IDbConnectionFactory _db;

    public DataSource Source => DataSource.S2000;

    public S2000Adapter(IDbConnectionFactory db) => _db = db;

    public async Task<IReadOnlyList<RiskMatch>> SearchAsync(
        RiskSearchContext context, CancellationToken ct)
    {
        // SQL query against S2000 database
        // Map results → RiskMatch
    }
}
```

## External API Adapter

```csharp
public class LloydsMDRAdapter : IRiskDataSourceAdapter
{
    private readonly HttpClient _http;
    private readonly DataSourceConfig _config;

    public DataSource Source => DataSource.LloydsMDR;

    public LloydsMDRAdapter(HttpClient http, DataSourceConfig config)
    {
        _http = http;
        _config = config;
    }

    public async Task<IReadOnlyList<RiskMatch>> SearchAsync(
        RiskSearchContext context, CancellationToken ct)
    {
        // 1. Map RiskSearchContext → Lloyd's API request format
        var apiRequest = MapToLloydsRequest(context);

        // 2. Call external API (with Polly retry/timeout)
        var response = await _http.PostAsJsonAsync(
            $"{_config.BaseUrl}/search", apiRequest, ct);

        // 3. Map Lloyd's API response → RiskMatch (common model)
        var lloydsResults = await response.Content
            .ReadFromJsonAsync<LloydsSearchResponse>(ct);

        return MapToRiskMatches(lloydsResults);
    }
}
```

---

## Orchestrator — Fan Out Across All Sources

```csharp
public class RiskSearchOrchestrator
{
    private readonly IEnumerable<IRiskDataSourceAdapter> _adapters;

    public RiskSearchOrchestrator(IEnumerable<IRiskDataSourceAdapter> adapters)
        => _adapters = adapters;

    public async Task<RiskClearanceResponse> SearchAsync(
        RiskClearanceRequest request, CancellationToken ct)
    {
        // 1. Build the common search context
        var context = new RiskSearchContext
        {
            InsuredDetails = request.InsuredDetails,
            TransactionTypes = request.SearchParameters.TransactionTypes,
            ClassOfBusiness = request.CoverageDetails.ClassOfBusiness,
            StatusFilter = request.SearchParameters.StatusFilter,
            LookBackMonths = request.SearchParameters.LookBackMonths,
            ConfidenceScore = request.SearchParameters.ConfidenceScore,
            NumberOfResults = request.SearchParameters.NumberOfResults
        };

        // 2. Find the adapters for requested data sources
        var requestedSources = request.SearchParameters.DataSources;
        var activeAdapters = _adapters
            .Where(a => requestedSources.Contains(a.Source));

        // 3. Fan out — call ALL sources in parallel
        var tasks = activeAdapters
            .Select(a => a.SearchAsync(context, ct));

        var results = await Task.WhenAll(tasks);

        // 4. Flatten, dedupe, rank by confidence, trim to numberOfResults
        var allMatches = results
            .SelectMany(r => r)
            .OrderByDescending(m => m.ConfidenceScore)
            .Take(request.SearchParameters.NumberOfResults)
            .ToList();

        return new RiskClearanceResponse
        {
            CorrelationId = request.CorrelationId,
            Matches = allMatches
        };
    }
}
```

---

## DI Registration

```csharp
// Program.cs or Startup.cs
builder.Services.AddScoped<IRiskDataSourceAdapter, S2000Adapter>();
builder.Services.AddScoped<IRiskDataSourceAdapter, MajescoAdapter>();
builder.Services.AddScoped<IRiskDataSourceAdapter, Rx0Adapter>();

// API-based sources — with typed HttpClient + Polly
builder.Services.AddHttpClient<LloydsMDRAdapter>(client =>
{
    client.BaseAddress = new Uri(config["DataSourceRegistry:LloydsMDR:BaseUrl"]);
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddPolicyHandler(GetRetryPolicy());

builder.Services.AddScoped<IRiskDataSourceAdapter, LloydsMDRAdapter>();
builder.Services.AddScoped<RiskSearchOrchestrator>();
```

---

## Flow Summary

```
Request comes in (same shape always)
        │
        ▼
  Orchestrator reads dataSources: ["S2000", "Majesco", "LloydsMDR"]
        │
        ├── S2000Adapter      → SQL query     → RiskMatch[]
        ├── MajescoAdapter    → SQL query     → RiskMatch[]
        └── LloydsMDRAdapter  → HTTP API call → RiskMatch[]
        │
        ▼
  Merge + Dedupe + Rank by confidence
        │
        ▼
  RiskClearanceResponse (common shape)
```

Adding a new API source tomorrow = write one adapter class + register in DI. 
The request model, response model, and orchestrator don't change.
