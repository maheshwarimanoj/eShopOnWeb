# Risk Clearance — Common Domain Model

## Problem

The current payload repeats the same `sourceSystemProviders` + `settings` block for every transaction type (Policy, Submission, Quote). Adding a new type or source means copy-pasting the entire block.

---

## Simplified Payload (Before → After)

### BEFORE (Current — ~95 lines, heavy duplication)

Each transaction type repeats the full provider + settings block.

### AFTER (Proposed — ~50 lines, zero duplication)

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

    "defaultProviderSettings": {
      "sourceProviderName": "GSIDataPlatform",
      "dataSources": ["S2000", "Majesco", "Rx0"],
      "confidenceScore": 50,
      "lookBackMonths": 6,
      "numberOfResults": 10,
      "statusFilter": ["Bound", "Cancelled"]
    },

    "transactionOverrides": {
      "Quote": {
        "lookBackMonths": 12,
        "statusFilter": ["Quoted", "Referred"]
      },
      "Submission": {
        "lookBackMonths": 3,
        "numberOfResults": 20
      }
    }
  }
}
```

**Key changes:**
- `transactionTypes` is now a simple array — just declare what you want
- `defaultProviderSettings` holds the shared config once
- `transactionOverrides` (optional) lets any type override specific settings
- Adding a new type = add one string to the array, optionally add overrides

---

## C# Domain Model

```csharp
namespace CededRe.RiskClearance.Domain;

/// <summary>
/// The canonical request model for Risk Clearance.
/// Any source system (Rx0, broker portal, future sources) maps into this.
/// </summary>
public sealed record RiskClearanceRequest
{
    public required string CorrelationId { get; init; }
    public required DateTimeOffset RequestTimestamp { get; init; }
    public required string SourceSystem { get; init; }
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
    public required IReadOnlyList<string> ClassOfBusiness { get; init; }
}

public sealed record SearchParameters
{
    /// <summary>
    /// Which transaction types to search: Policy, Quote, Submission, etc.
    /// New types just get added here — no structural changes needed.
    /// </summary>
    public required IReadOnlyList<string> TransactionTypes { get; init; }

    /// <summary>
    /// Shared settings applied to ALL transaction types by default.
    /// </summary>
    public required ProviderSettings DefaultProviderSettings { get; init; }

    /// <summary>
    /// Per-type overrides. Only the fields that differ need to be specified.
    /// Key = TransactionType name (e.g. "Quote").
    /// </summary>
    public IReadOnlyDictionary<string, ProviderSettingsOverride>? TransactionOverrides { get; init; }
}

public sealed record ProviderSettings
{
    public required string SourceProviderName { get; init; }
    public required IReadOnlyList<string> DataSources { get; init; }
    public required int ConfidenceScore { get; init; }
    public required int LookBackMonths { get; init; }
    public required int NumberOfResults { get; init; }
    public required IReadOnlyList<string> StatusFilter { get; init; }
}

/// <summary>
/// Nullable override — only set the fields you want to change for a specific type.
/// </summary>
public sealed record ProviderSettingsOverride
{
    public string? SourceProviderName { get; init; }
    public IReadOnlyList<string>? DataSources { get; init; }
    public int? ConfidenceScore { get; init; }
    public int? LookBackMonths { get; init; }
    public int? NumberOfResults { get; init; }
    public IReadOnlyList<string>? StatusFilter { get; init; }
}
```

---

## Resolved Settings Helper

At runtime, merge defaults + overrides per transaction type:

```csharp
public static class SettingsResolver
{
    /// <summary>
    /// For a given transaction type, returns the effective settings
    /// (defaults merged with any overrides).
    /// </summary>
    public static ProviderSettings Resolve(
        SearchParameters parameters,
        string transactionType)
    {
        var defaults = parameters.DefaultProviderSettings;

        if (parameters.TransactionOverrides is null ||
            !parameters.TransactionOverrides.TryGetValue(transactionType, out var overrides))
        {
            return defaults;
        }

        return defaults with
        {
            SourceProviderName = overrides.SourceProviderName ?? defaults.SourceProviderName,
            DataSources        = overrides.DataSources        ?? defaults.DataSources,
            ConfidenceScore    = overrides.ConfidenceScore     ?? defaults.ConfidenceScore,
            LookBackMonths     = overrides.LookBackMonths      ?? defaults.LookBackMonths,
            NumberOfResults    = overrides.NumberOfResults      ?? defaults.NumberOfResults,
            StatusFilter       = overrides.StatusFilter        ?? defaults.StatusFilter
        };
    }

    /// <summary>
    /// Returns resolved settings for ALL requested transaction types.
    /// </summary>
    public static IReadOnlyDictionary<string, ProviderSettings> ResolveAll(
        SearchParameters parameters)
    {
        return parameters.TransactionTypes
            .ToDictionary(
                type => type,
                type => Resolve(parameters, type));
    }
}
```

---

## Usage — Adding a New Source Tomorrow

**Scenario:** A new "Endorsement" transaction type needs to be searched with 24-month lookback.

**Change required in the request payload:**

```json
{
  "searchParameters": {
    "transactionTypes": ["Policy", "Submission", "Quote", "Endorsement"],
    "transactionOverrides": {
      "Endorsement": {
        "lookBackMonths": 24
      }
    }
  }
}
```

No code changes. No new classes. No structural changes to the API contract.

---

## Naming: "Transaction Type" vs Alternatives

| Term               | Pros                          | Cons                              |
|--------------------|-------------------------------|-----------------------------------|
| TransactionType    | Familiar in insurance domain  | Can confuse with DB transactions  |
| SearchScope        | Clear intent (what to search) | Less domain-specific              |
| RiskSearchCategory | Very explicit                 | Verbose                           |

**Recommendation:** Use `TransactionType` — it's the standard London Market term for Policy/Quote/Submission classification. Your instinct was right.

---

## Enum Alternative (Strongly Typed)

If you want compile-time safety instead of raw strings:

```csharp
[JsonConverter(typeof(JsonStringEnumConverter))]
public enum TransactionType
{
    Policy,
    Quote,
    Submission,
    Endorsement,
    Renewal,
    Claim  // future-proof
}
```

Then change `IReadOnlyList<string>` → `IReadOnlyList<TransactionType>` and the dictionary key to `TransactionType`. Trade-off: safer but requires a code deploy to add new types vs. strings which are config-driven.
