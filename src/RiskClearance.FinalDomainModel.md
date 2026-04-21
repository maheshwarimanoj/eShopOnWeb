# Risk Clearance — Common Domain Model (Corrected)

## Key Relationship

```
Provider (e.g. GSIDataPlatform)
   └── owns DataSources (S2000, Majesco, Rx0)

Provider (e.g. LloydsMDR)
   └── owns DataSources (Eclipse, Genius)
```

Data sources are scoped to a provider. The model must preserve this.

---

## Simplified JSON Payload

```json
{
  "correlationId": "RC-20260317-000123",
  "requestTimestamp": "2026-03-17T12:45:00Z",
  "sourceSystem": "Rx0",
  "sourceSystemReference": "RX0-REQ-0001",

  "insuredDetails": [
    {
      "insuredName": "CocaCola Inc",
      "dbaAka": "GTC Logistics",
      "uniqueId": "123456789",
      "countryCode": "US"
    }
  ],

  "coverageDetails": {
    "classOfBusiness": ["PVT"]
  },

  "searchParameters": {
    "transactionTypes": ["Policy", "Submission", "Quote"],
    "providers": [
      {
        "name": "GSIDataPlatform",
        "dataSources": ["S2000", "Majesco", "Rx0"],
        "confidenceScore": 50,
        "lookBackMonths": 6,
        "numberOfResults": 10,
        "statusFilter": ["Bound", "Cancelled"]
      }
    ]
  }
}
```

**What's different from your original:**
- `transactionTypes` pulled out as a simple array (no longer repeated per type)
- `providers` is a clean array — each provider carries its own sources and settings
- No duplication — everything declared once

**Adding a second provider tomorrow:**

```json
{
  "searchParameters": {
    "transactionTypes": ["Policy", "Quote"],
    "providers": [
      {
        "name": "GSIDataPlatform",
        "dataSources": ["S2000", "Majesco"],
        "confidenceScore": 50,
        "lookBackMonths": 6,
        "numberOfResults": 10,
        "statusFilter": ["Bound", "Cancelled"]
      },
      {
        "name": "LloydsMDR",
        "dataSources": ["Eclipse", "Genius"],
        "confidenceScore": 60,
        "lookBackMonths": 12,
        "numberOfResults": 5,
        "statusFilter": ["Bound", "InForce"]
      }
    ]
  }
}
```

Each provider owns its own sources and settings. Clean separation.

---

## Enums

```csharp
using System.Text.Json.Serialization;

namespace CededRe.RiskClearance.Domain.Enums;

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum TransactionType
{
    Policy,
    Quote,
    Submission,
    Endorsement,
    Renewal,
    Claim
}

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum SourceProvider
{
    GSIDataPlatform,
    LloydsMDR,
    BrokerDirect
}

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum DataSource
{
    // GSIDataPlatform sources
    S2000,
    Majesco,
    Rx0,

    // LloydsMDR sources
    Eclipse,
    Genius,

    // Future
    BrokerHub
}

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum TransactionStatus
{
    Bound,
    Cancelled,
    Quoted,
    Referred,
    Declined,
    NotTakenUp,
    Expired,
    InForce
}

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum SourceSystem
{
    Rx0,
    BrokerPortal,
    Underwriter,
    ThirdParty
}

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum ClassOfBusiness
{
    PVT,
    XX,
    YY,
    PropertyDirect,
    CasualtyTreaty,
    MarineHull,
    AviationLiability
}
```

---

## C# Domain Model

```csharp
namespace CededRe.RiskClearance.Domain;

public sealed record RiskClearanceRequest
{
    public required string CorrelationId { get; init; }
    public required DateTimeOffset RequestTimestamp { get; init; }
    public required SourceSystem SourceSystem { get; init; }
    public required string SourceSystemReference { get; init; }
    public required IReadOnlyList<InsuredDetail> InsuredDetails { get; init; }
    public required CoverageDetail CoverageDetails { get; init; }
    public required SearchParameters SearchParameters { get; init; }
}

public sealed record InsuredDetail
{
    public required string InsuredName { get; init; }
    public string? DbaAka { get; init; }
    public required string UniqueId { get; init; }
    public required string CountryCode { get; init; }
}

public sealed record CoverageDetail
{
    public required IReadOnlyList<ClassOfBusiness> ClassOfBusiness { get; init; }
}

public sealed record SearchParameters
{
    public required IReadOnlyList<TransactionType> TransactionTypes { get; init; }
    public required IReadOnlyList<ProviderConfig> Providers { get; init; }
}

/// <summary>
/// Each provider carries its own data sources and search settings.
/// Settings are scoped to the provider because different providers
/// may have different capabilities, SLAs, and data retention.
/// </summary>
public sealed record ProviderConfig
{
    public required SourceProvider Name { get; init; }
    public required IReadOnlyList<DataSource> DataSources { get; init; }
    public required int ConfidenceScore { get; init; }
    public required int LookBackMonths { get; init; }
    public required int NumberOfResults { get; init; }
    public required IReadOnlyList<TransactionStatus> StatusFilter { get; init; }
}
```

---

## Adapter Pattern (Updated)

```csharp
/// <summary>
/// One adapter per PROVIDER (not per data source).
/// The provider knows how to query its own sources.
/// </summary>
public interface IProviderAdapter
{
    SourceProvider Provider { get; }
    Task<IReadOnlyList<RiskMatch>> SearchAsync(
        ProviderSearchContext context,
        CancellationToken ct);
}

/// <summary>
/// Context passed to a provider adapter — includes which of its
/// sources to search and the search criteria.
/// </summary>
public sealed record ProviderSearchContext
{
    public required IReadOnlyList<InsuredDetail> InsuredDetails { get; init; }
    public required IReadOnlyList<TransactionType> TransactionTypes { get; init; }
    public required IReadOnlyList<ClassOfBusiness> ClassOfBusiness { get; init; }
    public required IReadOnlyList<DataSource> DataSources { get; init; }
    public required IReadOnlyList<TransactionStatus> StatusFilter { get; init; }
    public required int LookBackMonths { get; init; }
    public required int ConfidenceScore { get; init; }
    public required int NumberOfResults { get; init; }
}

public sealed record RiskMatch
{
    public required string MatchId { get; init; }
    public required SourceProvider Provider { get; init; }
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

## Orchestrator

```csharp
public class RiskSearchOrchestrator
{
    private readonly IEnumerable<IProviderAdapter> _adapters;

    public async Task<RiskClearanceResponse> SearchAsync(
        RiskClearanceRequest request, CancellationToken ct)
    {
        var tasks = request.SearchParameters.Providers
            .Select(providerConfig =>
            {
                // Find the adapter for this provider
                var adapter = _adapters.First(
                    a => a.Provider == providerConfig.Name);

                // Build context from request + provider-specific config
                var context = new ProviderSearchContext
                {
                    InsuredDetails  = request.InsuredDetails,
                    TransactionTypes = request.SearchParameters.TransactionTypes,
                    ClassOfBusiness = request.CoverageDetails.ClassOfBusiness,
                    DataSources     = providerConfig.DataSources,
                    StatusFilter    = providerConfig.StatusFilter,
                    LookBackMonths  = providerConfig.LookBackMonths,
                    ConfidenceScore = providerConfig.ConfidenceScore,
                    NumberOfResults = providerConfig.NumberOfResults
                };

                return adapter.SearchAsync(context, ct);
            });

        var results = await Task.WhenAll(tasks);

        return new RiskClearanceResponse
        {
            CorrelationId = request.CorrelationId,
            Matches = results
                .SelectMany(r => r)
                .OrderByDescending(m => m.ConfidenceScore)
                .ToList()
        };
    }
}
```

---

## Flow

```
Request
  │
  ├── transactionTypes: [Policy, Quote]     ← WHAT to search
  │
  └── providers:                            ← WHERE to search
       ├── GSIDataPlatform                
       │    ├── sources: [S2000, Majesco]   ← provider's own sources
       │    └── settings: 6mo, score>50
       │
       └── LloydsMDR                      
            ├── sources: [Eclipse]          ← different sources
            └── settings: 12mo, score>60
```

```
Orchestrator
  │
  ├── GSIDataPlatformAdapter.SearchAsync()  → RiskMatch[]
  └── LloydsMDRAdapter.SearchAsync()        → RiskMatch[]
  │
  ▼
  Merge + Rank → RiskClearanceResponse
```

---

## Comparison: Original vs Common Domain Model

| Aspect              | Original                           | Common Model                   |
|---------------------|------------------------------------|---------------------------------|
| Transaction types   | Repeated block per type            | Simple enum array               |
| Provider + sources  | Nested 3 levels deep, duplicated   | One `ProviderConfig` per provider|
| Settings            | Duplicated per type × per provider | Once per provider               |
| Adding new type     | Copy-paste entire block            | Add to enum array               |
| Adding new provider | Major structural change            | Add one ProviderConfig + adapter|
| Lines of JSON       | ~95                                | ~30                             |
