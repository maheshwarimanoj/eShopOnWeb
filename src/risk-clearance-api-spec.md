# Risk Clearance API — OpenAPI 3.0.3 Specification

```yaml
openapi: 3.0.3
info:
  title: Risk Clearance API
  description: |
    The Risk Clearance API enables source systems to search for risk data 
    across multiple providers and data sources using a common domain model.

    ## Key Concepts

    - **Source System**: The system originating the request (e.g. Rx0, BrokerPortal)
    - **Provider**: A data platform that owns one or more data sources (e.g. GSIDataPlatform, LloydsMDR)
    - **Data Source**: A specific system within a provider (e.g. S2000, Majesco, Eclipse)
    - **Transaction Type**: The category of risk record being searched (Policy, Quote, Submission)

    ## Common Domain Model

    The request payload follows a common domain model regardless of whether the 
    underlying provider is an internal database or an external API. Provider 
    connection details (URLs, auth, timeouts) are managed server-side via configuration 
    and are never exposed in the request contract.

    ## Authentication

    All requests require:
    - JWT Bearer token (Azure AD or B2C)
    - APIM Subscription Key (`Ocp-Apim-Subscription-Key` header)
  version: 1.0.0
  contact:
    name: Ceded-Re Engineering Team

servers:
  - url: https://api.mrsgm.com/ceded-re/v1
    description: Production
  - url: https://api-uat.mrsgm.com/ceded-re/v1
    description: UAT

security:
  - bearerAuth: []
    subscriptionKey: []

paths:
  /risk-clearance:
    post:
      operationId: searchRiskClearance
      summary: Search for risk clearance data
      description: |
        Searches for risk data across one or more providers and their data sources.
        
        The caller specifies:
        - **What** to search for — insured details, coverage, transaction types
        - **Where** to search — which providers and data sources
        - **How** to filter — confidence threshold, lookback period, status filters
        
        The system resolves how to connect to each provider (DB or API) 
        from server-side configuration. The request shape is identical 
        regardless of the provider type.
      tags:
        - Risk Clearance
      parameters:
        - $ref: '#/components/parameters/CorrelationIdHeader'
        - $ref: '#/components/parameters/SubscriptionKeyHeader'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RiskClearanceRequest'
            examples:
              singleProvider:
                summary: Single provider (internal DB)
                description: |
                  Standard request searching GSIDataPlatform's internal 
                  data sources for Policy, Submission, and Quote records.
                value:
                  correlationId: "RC-20260317-000123"
                  requestTimestamp: "2026-03-17T12:45:00Z"
                  sourceSystem: "Rx0"
                  sourceSystemReference: "RX0-REQ-0001"
                  insuredDetails:
                    - insuredName: "CocaCola Inc"
                      dbaAka: "GTC Logistics"
                      uniqueId: "123456789"
                      countryCode: "US"
                    - insuredName: "CocaCola Tradings"
                      dbaAka: "GTC Tradings"
                      uniqueId: "987654321"
                      countryCode: "IN"
                  coverageDetails:
                    classOfBusiness:
                      - "PVT"
                      - "XX"
                      - "YY"
                  searchParameters:
                    transactionTypes:
                      - "Policy"
                      - "Submission"
                      - "Quote"
                    providers:
                      - name: "GSIDataPlatform"
                        dataSources:
                          - "S2000"
                          - "Majesco"
                          - "Rx0"
                        confidenceScore: 50
                        lookBackMonths: 6
                        numberOfResults: 10
                        statusFilter:
                          - "Bound"
                          - "Cancelled"

              multipleProviders:
                summary: Multiple providers (internal DB + external API)
                description: |
                  Request searching across both GSIDataPlatform (internal) 
                  and LloydsMDR (external API). The request shape is 
                  identical — the system resolves connection details 
                  from server-side configuration.
                value:
                  correlationId: "RC-20260317-000456"
                  requestTimestamp: "2026-03-17T14:30:00Z"
                  sourceSystem: "BrokerPortal"
                  sourceSystemReference: "BP-REQ-0042"
                  insuredDetails:
                    - insuredName: "Acme Corp"
                      dbaAka: "Acme International"
                      uniqueId: "555666777"
                      countryCode: "GB"
                  coverageDetails:
                    classOfBusiness:
                      - "MarineHull"
                  searchParameters:
                    transactionTypes:
                      - "Policy"
                      - "Quote"
                    providers:
                      - name: "GSIDataPlatform"
                        dataSources:
                          - "S2000"
                          - "Majesco"
                        confidenceScore: 50
                        lookBackMonths: 6
                        numberOfResults: 10
                        statusFilter:
                          - "Bound"
                          - "Cancelled"
                      - name: "LloydsMDR"
                        dataSources:
                          - "Eclipse"
                          - "Genius"
                        confidenceScore: 60
                        lookBackMonths: 12
                        numberOfResults: 5
                        statusFilter:
                          - "Bound"
                          - "InForce"

              minimalRequest:
                summary: Minimal request (single insured, single provider)
                description: |
                  Simplest possible request — one insured, one provider, 
                  one data source, one transaction type.
                value:
                  correlationId: "RC-20260317-000789"
                  requestTimestamp: "2026-03-17T16:00:00Z"
                  sourceSystem: "Rx0"
                  sourceSystemReference: "RX0-REQ-0099"
                  insuredDetails:
                    - insuredName: "Tesla Inc"
                      uniqueId: "111222333"
                      countryCode: "US"
                  coverageDetails:
                    classOfBusiness:
                      - "PVT"
                  searchParameters:
                    transactionTypes:
                      - "Policy"
                    providers:
                      - name: "GSIDataPlatform"
                        dataSources:
                          - "S2000"
                        confidenceScore: 50
                        lookBackMonths: 6
                        numberOfResults: 10
                        statusFilter:
                          - "Bound"

      responses:
        '200':
          description: Risk clearance results returned successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RiskClearanceResponse'
              examples:
                successResponse:
                  summary: Successful response with matches from multiple providers
                  value:
                    correlationId: "RC-20260317-000456"
                    responseTimestamp: "2026-03-17T14:30:02Z"
                    providerResults:
                      - providerName: "GSIDataPlatform"
                        status: "Success"
                        matches:
                          - matchId: "GSI-M-00001"
                            dataSource: "S2000"
                            transactionType: "Policy"
                            insuredName: "Acme Corp"
                            confidenceScore: 85.5
                            status: "Bound"
                            effectiveDate: "2025-09-15"
                            expiryDate: "2026-09-15"
                            policyReference: "POL-2025-44821"
                          - matchId: "GSI-M-00002"
                            dataSource: "Majesco"
                            transactionType: "Quote"
                            insuredName: "Acme International"
                            confidenceScore: 72.0
                            status: "Quoted"
                            effectiveDate: "2026-01-01"
                            quoteReference: "QTE-2026-10234"
                      - providerName: "LloydsMDR"
                        status: "Success"
                        matches:
                          - matchId: "LMD-M-00001"
                            dataSource: "Eclipse"
                            transactionType: "Policy"
                            insuredName: "Acme Corp"
                            confidenceScore: 91.2
                            status: "Bound"
                            effectiveDate: "2024-06-01"
                            expiryDate: "2025-06-01"
                            policyReference: "ECL-POL-88123"

                partialFailure:
                  summary: Partial success — one provider failed
                  description: |
                    GSIDataPlatform returned results but LloydsMDR 
                    timed out. The response clearly indicates which 
                    provider failed and why.
                  value:
                    correlationId: "RC-20260317-000456"
                    responseTimestamp: "2026-03-17T14:30:05Z"
                    providerResults:
                      - providerName: "GSIDataPlatform"
                        status: "Success"
                        matches:
                          - matchId: "GSI-M-00001"
                            dataSource: "S2000"
                            transactionType: "Policy"
                            insuredName: "Acme Corp"
                            confidenceScore: 85.5
                            status: "Bound"
                            effectiveDate: "2025-09-15"
                            expiryDate: "2026-09-15"
                            policyReference: "POL-2025-44821"
                      - providerName: "LloydsMDR"
                        status: "Failed"
                        errorCode: "PROVIDER_TIMEOUT"
                        errorMessage: "LloydsMDR did not respond within 30 seconds"
                        matches: []

        '400':
          description: Invalid request payload
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                correlationId: "RC-20260317-000123"
                errorCode: "VALIDATION_ERROR"
                errors:
                  - field: "insuredDetails"
                    message: "At least one insured detail is required"
                  - field: "searchParameters.providers"
                    message: "At least one provider must be specified"

        '401':
          description: Missing or invalid JWT token
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                correlationId: "RC-20260317-000123"
                errorCode: "UNAUTHORIZED"
                errors:
                  - field: "Authorization"
                    message: "Bearer token is missing or expired"

        '429':
          description: Rate limit exceeded
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                correlationId: "RC-20260317-000123"
                errorCode: "RATE_LIMIT_EXCEEDED"
                errors:
                  - field: "request"
                    message: "Rate limit of 100 requests per minute exceeded"

        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  # ──────────────────────────────────────────────
  # Security Schemes
  # ──────────────────────────────────────────────
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: Azure AD or B2C JWT token
    subscriptionKey:
      type: apiKey
      in: header
      name: Ocp-Apim-Subscription-Key
      description: APIM subscription key

  # ──────────────────────────────────────────────
  # Parameters
  # ──────────────────────────────────────────────
  parameters:
    CorrelationIdHeader:
      name: X-Correlation-Id
      in: header
      required: false
      description: Optional correlation ID for distributed tracing. If not provided, the value from the request body is used.
      schema:
        type: string
        example: "RC-20260317-000123"

    SubscriptionKeyHeader:
      name: Ocp-Apim-Subscription-Key
      in: header
      required: true
      description: APIM subscription key for the Risk Clearance API product
      schema:
        type: string

  # ──────────────────────────────────────────────
  # Request Schemas
  # ──────────────────────────────────────────────
  schemas:
    RiskClearanceRequest:
      type: object
      description: |
        Common domain model for risk clearance search requests. 
        This shape is identical regardless of whether the underlying 
        providers are internal databases or external APIs.
      required:
        - correlationId
        - requestTimestamp
        - sourceSystem
        - sourceSystemReference
        - insuredDetails
        - coverageDetails
        - searchParameters
      properties:
        correlationId:
          type: string
          description: Unique identifier for this request, used for tracing and idempotency
          example: "RC-20260317-000123"
          pattern: '^RC-\d{8}-\d{6}$'

        requestTimestamp:
          type: string
          format: date-time
          description: ISO 8601 timestamp of when the request was created
          example: "2026-03-17T12:45:00Z"

        sourceSystem:
          type: string
          description: The system originating this request
          enum:
            - Rx0
            - BrokerPortal
            - Underwriter
            - ThirdParty
          example: "Rx0"

        sourceSystemReference:
          type: string
          description: The originating system's own reference for this request
          example: "RX0-REQ-0001"

        insuredDetails:
          type: array
          description: One or more insured entities to search for. Multiple entries enable searching across related entities (e.g. parent company and subsidiary).
          minItems: 1
          items:
            $ref: '#/components/schemas/InsuredDetail'

        coverageDetails:
          $ref: '#/components/schemas/CoverageDetail'

        searchParameters:
          $ref: '#/components/schemas/SearchParameters'

    # ── Insured Detail ──
    InsuredDetail:
      type: object
      description: Details of an insured entity to search for
      required:
        - insuredName
        - uniqueId
        - countryCode
      properties:
        insuredName:
          type: string
          description: Legal name of the insured entity
          example: "CocaCola Inc"

        dbaAka:
          type: string
          description: Trading name or 'also known as' name
          example: "GTC Logistics"
          nullable: true

        uniqueId:
          type: string
          description: Unique identifier for the insured (e.g. company registration number, DUNS number)
          example: "123456789"

        countryCode:
          type: string
          description: ISO 3166-1 alpha-2 country code of the insured's domicile
          pattern: '^[A-Z]{2}$'
          example: "US"

    # ── Coverage Detail ──
    CoverageDetail:
      type: object
      description: Coverage information to narrow the risk search
      required:
        - classOfBusiness
      properties:
        classOfBusiness:
          type: array
          description: One or more classes of business to search within
          minItems: 1
          items:
            type: string
            enum:
              - PVT
              - XX
              - YY
              - PropertyDirect
              - CasualtyTreaty
              - MarineHull
              - AviationLiability
          example: ["PVT", "XX"]

    # ── Search Parameters ──
    SearchParameters:
      type: object
      description: |
        Defines what to search for (transaction types) and where to 
        search (providers with their data sources and settings).
      required:
        - transactionTypes
        - providers
      properties:
        transactionTypes:
          type: array
          description: |
            Which types of risk transactions to search for. 
            This is a shared filter applied across all providers.
          minItems: 1
          items:
            type: string
            enum:
              - Policy
              - Quote
              - Submission
              - Endorsement
              - Renewal
              - Claim
          example: ["Policy", "Submission", "Quote"]

        providers:
          type: array
          description: |
            One or more data providers to search. Each provider 
            carries its own data sources and search settings because 
            different providers may have different data retention, 
            SLAs, and capabilities. The system resolves connection 
            details (DB or API) from server-side configuration.
          minItems: 1
          items:
            $ref: '#/components/schemas/ProviderConfig'

    # ── Provider Config ──
    ProviderConfig:
      type: object
      description: |
        Configuration for a single data provider. The provider name 
        maps to server-side configuration that determines how to 
        connect (internal DB, external REST API, SOAP service, etc.). 
        The caller only specifies which provider and sources to use, 
        plus search settings — never connection details.
      required:
        - name
        - dataSources
        - confidenceScore
        - lookBackMonths
        - numberOfResults
        - statusFilter
      properties:
        name:
          type: string
          description: |
            Name of the data provider. Must match a registered 
            provider in server-side configuration.
          enum:
            - GSIDataPlatform
            - LloydsMDR
            - SwissReCatNet
            - BrokerDirect
          example: "GSIDataPlatform"

        dataSources:
          type: array
          description: |
            Specific data sources within this provider to search. 
            These are scoped to the provider — S2000, Majesco, Rx0 
            belong to GSIDataPlatform; Eclipse, Genius belong to 
            LloydsMDR.
          minItems: 1
          items:
            type: string
            enum:
              - S2000
              - Majesco
              - Rx0
              - Eclipse
              - Genius
              - CatNetAPI
              - BrokerHub
          example: ["S2000", "Majesco", "Rx0"]

        confidenceScore:
          type: integer
          description: Minimum confidence score threshold (0-100). Only matches at or above this score are returned.
          minimum: 0
          maximum: 100
          example: 50

        lookBackMonths:
          type: integer
          description: How many months back to search from the request timestamp
          minimum: 1
          maximum: 120
          example: 6

        numberOfResults:
          type: integer
          description: Maximum number of results to return from this provider
          minimum: 1
          maximum: 100
          example: 10

        statusFilter:
          type: array
          description: Only return transactions with these statuses
          minItems: 1
          items:
            type: string
            enum:
              - Bound
              - Cancelled
              - Quoted
              - Referred
              - Declined
              - NotTakenUp
              - Expired
              - InForce
          example: ["Bound", "Cancelled"]

    # ──────────────────────────────────────────────
    # Response Schemas
    # ──────────────────────────────────────────────
    RiskClearanceResponse:
      type: object
      description: |
        Response containing risk clearance results grouped by provider. 
        Each provider reports its own status independently, so a partial 
        failure (e.g. one provider times out) still returns results from 
        the other providers.
      required:
        - correlationId
        - responseTimestamp
        - providerResults
      properties:
        correlationId:
          type: string
          description: Echoed from the request for tracing
          example: "RC-20260317-000123"

        responseTimestamp:
          type: string
          format: date-time
          description: ISO 8601 timestamp of when the response was generated
          example: "2026-03-17T12:45:02Z"

        providerResults:
          type: array
          description: Results grouped by provider, each with its own success/failure status
          items:
            $ref: '#/components/schemas/ProviderResult'

    # ── Provider Result ──
    ProviderResult:
      type: object
      description: Results from a single provider
      required:
        - providerName
        - status
        - matches
      properties:
        providerName:
          type: string
          description: Name of the provider that returned these results
          example: "GSIDataPlatform"

        status:
          type: string
          description: Whether this provider returned results successfully
          enum:
            - Success
            - Failed
            - PartialSuccess
          example: "Success"

        errorCode:
          type: string
          description: Error code if the provider failed (only present when status is Failed)
          nullable: true
          example: "PROVIDER_TIMEOUT"

        errorMessage:
          type: string
          description: Human-readable error message (only present when status is Failed)
          nullable: true
          example: "LloydsMDR did not respond within 30 seconds"

        matches:
          type: array
          description: Risk matches found by this provider
          items:
            $ref: '#/components/schemas/RiskMatch'

    # ── Risk Match ──
    RiskMatch:
      type: object
      description: A single risk match returned by a provider
      required:
        - matchId
        - dataSource
        - transactionType
        - insuredName
        - confidenceScore
        - status
        - effectiveDate
      properties:
        matchId:
          type: string
          description: Unique identifier for this match within the provider
          example: "GSI-M-00001"

        dataSource:
          type: string
          description: Which data source within the provider produced this match
          example: "S2000"

        transactionType:
          type: string
          description: The type of transaction this match represents
          enum:
            - Policy
            - Quote
            - Submission
            - Endorsement
            - Renewal
            - Claim
          example: "Policy"

        insuredName:
          type: string
          description: Insured name as found in the source system
          example: "CocaCola Inc"

        confidenceScore:
          type: number
          format: double
          description: How confident the match is (0.0 - 100.0)
          example: 85.5

        status:
          type: string
          description: Current status of the transaction
          enum:
            - Bound
            - Cancelled
            - Quoted
            - Referred
            - Declined
            - NotTakenUp
            - Expired
            - InForce
          example: "Bound"

        effectiveDate:
          type: string
          format: date
          description: Effective date of the policy/quote/submission
          example: "2025-09-15"

        expiryDate:
          type: string
          format: date
          description: Expiry date (applicable for policies)
          nullable: true
          example: "2026-09-15"

        policyReference:
          type: string
          description: Policy reference number (if transaction type is Policy)
          nullable: true
          example: "POL-2025-44821"

        quoteReference:
          type: string
          description: Quote reference number (if transaction type is Quote)
          nullable: true
          example: "QTE-2026-10234"

        submissionReference:
          type: string
          description: Submission reference number (if transaction type is Submission)
          nullable: true
          example: "SUB-2026-00512"

        additionalData:
          type: object
          description: Provider-specific additional data that doesn't fit the common model
          additionalProperties: true
          nullable: true

    # ── Error Response ──
    ErrorResponse:
      type: object
      description: Standard error response
      required:
        - correlationId
        - errorCode
        - errors
      properties:
        correlationId:
          type: string
          description: Echoed from the request for tracing
          example: "RC-20260317-000123"

        errorCode:
          type: string
          description: Machine-readable error category
          enum:
            - VALIDATION_ERROR
            - UNAUTHORIZED
            - FORBIDDEN
            - RATE_LIMIT_EXCEEDED
            - INTERNAL_ERROR
          example: "VALIDATION_ERROR"

        errors:
          type: array
          description: Detailed list of errors
          items:
            type: object
            required:
              - field
              - message
            properties:
              field:
                type: string
                description: Which field caused the error
                example: "insuredDetails"
              message:
                type: string
                description: Human-readable error description
                example: "At least one insured detail is required"

tags:
  - name: Risk Clearance
    description: |
      Risk clearance search operations. Enables source systems to 
      search for existing risk data across multiple providers and 
      data sources using a common domain model.
```
