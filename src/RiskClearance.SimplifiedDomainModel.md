# Risk Clearance — Simplified Common Domain Model

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
    "classOfBusiness": ["PVT", "XX", "YY"]
  },

  "searchParameters": {
    "transactionTypes": ["Policy", "Submission", "Quote"],
    "sourceProviderName": "GSIDataPlatform",
    "dataSources": ["S2000", "Majesco", "Rx0"],
    "confidenceScore": 50,
    "lookBackMonths": 6,
    "numberOfResults": 10,
    "statusFilter": ["Bound", "Cancelled"]
  }
}
```

No nesting, no overrides, no repeated blocks. Just flat, clean config.

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
    LloydsMDR,        // future
    BrokerDirect      // future
}

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum DataSource
{
    S2000,
    Majesco,
    Rx0,
    Eclipse,          // future
    Genius            // future
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

## C# Domain Model (Flat & Simple)

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
    public required string CountryCode { get; init; }  // ISO 3166-1 alpha-2
}

public sealed record CoverageDetail
{
    public required IReadOnlyList<ClassOfBusiness> ClassOfBusiness { get; init; }
}

public sealed record SearchParameters
{
    public required IReadOnlyList<TransactionType> TransactionTypes { get; init; }
    public required SourceProvider SourceProviderName { get; init; }
    public required IReadOnlyList<DataSource> DataSources { get; init; }
    public required int ConfidenceScore { get; init; }
    public required int LookBackMonths { get; init; }
    public required int NumberOfResults { get; init; }
    public required IReadOnlyList<TransactionStatus> StatusFilter { get; init; }
}
```

---

## What Changed vs Original

| Aspect              | Original Payload          | Common Domain Model         |
|---------------------|---------------------------|-----------------------------|
| Transaction types   | 3 separate nested blocks  | Simple enum array           |
| Provider settings   | Repeated per type         | Declared once, flat         |
| Data sources        | Raw strings               | Strongly typed enum         |
| Status filters      | Raw strings               | Strongly typed enum         |
| Lines of JSON       | ~95                       | ~25                         |
| Adding new type     | Copy-paste entire block   | Add one value to array      |
| Adding new source   | New provider block        | Add enum value + config     |

---

## Adding a New Source Tomorrow

Say a new **LloydsMDR** provider comes in. All you do:

1. Add `LloydsMDR` to the `SourceProvider` enum (already there above)
2. Caller sends:

```json
{
  "searchParameters": {
    "transactionTypes": ["Policy", "Quote"],
    "sourceProviderName": "LloydsMDR",
    "dataSources": ["Eclipse"],
    "confidenceScore": 60,
    "lookBackMonths": 12,
    "numberOfResults": 5,
    "statusFilter": ["Bound", "InForce"]
  }
}
```

Same model, different values. No structural changes.
