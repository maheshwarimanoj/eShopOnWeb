# Risk Clearance Domain API — OpenAPI 3.0.3 Specification

```yaml
openapi: 3.0.3
info:
  title: Risk Clearance Domain API
  version: 1.0.0
  description: Search and retrieve risk clearance data across providers (risk clearance domain)

servers:
  - url: https://api.example.com/v1

tags:
  - name: Risks

paths:
  /risks:
    post:
      tags: [Risks]
      summary: Search for risk clearance data
      operationId: searchRisks
      parameters:
        - name: X-Entity
          in: header
          required: true
          schema: { type: string }
          description: Entity identifier for the request
          example: "MRSI"
        - name: X-Product
          in: header
          required: true
          schema: { type: string }
          description: Product identifier
          example: "Casualty"
        - name: X-SourceId
          in: header
          required: true
          schema: { type: string }
          description: Source system identifier
          example: "Rx0"
        - name: X-SourceInfo
          in: header
          required: false
          schema: { type: string }
          description: Additional source metadata
          example: "Rx0:RiskClearanceSearch"
        - name: X-CorrelationId
          in: header
          required: false
          schema: { type: string }
          description: Source System Correlation ID
        - name: X-Provider
          in: header
          required: true
          schema: { type: string }
          description: Data provider to search. The system resolves connection details (internal DB or external API) from server-side configuration.
          example: "GSIDataPlatform"
        - name: traceparent
          in: header
          required: false
          schema: { type: string }
          description: HTTP header field defined by the W3C Trace Context specification that identifies a request within a distributed tracing system.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              # RiskClearanceSearch (inline)
              type: object
              properties:
                insuredDetails:
                  type: array
                  description: One or more insured entities to search for. Multiple entries enable searching across related entities (e.g. parent company and subsidiary).
                  items:
                    type: object  # InsuredDetail
                    properties:
                      insured:
                        type: object  # Insured
                        properties:
                          type:
                            type: string
                            enum: [Primary, Secondary, Additional]
                            example: Primary
                          entityName: { type: string }
                          entityType:
                            type: string
                            description: Legal form of insured entity.
                            example: Corporation
                          stateOfDomicile:
                            type: string
                            description: State/Province of domicile for the insured entity.
                            example: IL
                          dnbNumber:
                            type: string
                            description: "Dun & Bradstreet D-U-N-S number (9 digits) for the insured."
                            pattern: '^\d{9}$'
                            example: "123456789"
                          countryCode:
                            type: string
                            description: ISO 3166-1 alpha-2 country code of the insured's domicile.
                            pattern: '^[A-Z]{2}$'
                            example: "US"
                          identifiers:
                            type: array
                            description: Typed identifiers for the insured entity (DUNS, LEI, CompaniesHouse, etc.).
                            items:
                              type: object
                              properties:
                                type:
                                  type: string
                                  enum: [DUNS, LEI, CompaniesHouse, Internal]
                                  example: DUNS
                                value: { type: string, example: "123456789" }
                              additionalProperties: false
                          aliases:
                            type: array
                            description: Trading names or also-known-as names for the insured.
                            items: { type: string }
                            example: ["GTC Logistics", "CocaCola Bottling Co"]
                          addresses:
                            type: array
                            items:
                              type: object  # Address
                              properties:
                                type:
                                  type: string
                                  enum: [Primary, Secondary, Billing, Mailing]
                                  example: Primary
                                addressLine1: { type: string }
                                addressLine2: { type: string }
                                city:         { type: string }
                                region:
                                  type: string
                                  description: State/Province/Region code
                                  example: IL
                                state:   { type: string, example: IL }
                                zipCode: { type: string, example: "60061" }
                                country: { type: string, example: US }
                                county:  { type: string, example: Cook }
                              additionalProperties: false
                          industryCode: { type: string, example: "111120" }
                          webAddress: { type: string }
                        additionalProperties: false
                    additionalProperties: false
                coverageDetails:
                  type: object
                  properties:
                    lineOfBusiness:      { type: string, example: ESCasualty }
                    subLineOfBusiness:   { type: string, example: "LiquorLiability" }
                    classOfBusiness:
                      type: array
                      description: One or more classes of business to search within.
                      items:
                        type: string
                        enum:
                          - PVT
                          - PropertyDirect
                          - CasualtyTreaty
                          - MarineHull
                          - AviationLiability
                          - ESCasualty
                      example: ["ESCasualty"]
                  additionalProperties: false
                searchCriteria:
                  type: array
                  description: Each entry specifies a transaction type and its associated search settings. Different transaction types may require different search periods, confidence thresholds, result limits, and status filters.
                  items:
                    type: object  # SearchCriteriaEntry
                    properties:
                      transactionType:
                        type: string
                        description: The category of risk record to search for.
                        enum:
                          - Policy
                          - Quote
                          - Submission
                          - Endorsement
                          - Renewal
                          - Claim
                        example: Policy
                      confidenceScore:
                        type: integer
                        description: Minimum confidence threshold (0-100) defined by the source provider. Passed through to the data provider as a filter.
                        minimum: 0
                        maximum: 100
                        example: 50
                      searchPeriodMonths:
                        type: integer
                        description: How many months back to search from the request timestamp.
                        minimum: 1
                        maximum: 120
                        example: 6
                      numberOfResults:
                        type: integer
                        description: Maximum number of results to return for this transaction type.
                        minimum: 1
                        maximum: 100
                        example: 10
                      statusFilter:
                        type: array
                        description: Only return transactions with these statuses.
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
                            - Received
                            - Cleared
                            - NotCleared
                            - NotReviewed
                        example: ["Bound", "Cancelled"]
                    additionalProperties: false
              additionalProperties: false
            examples:
              singleTransactionType:
                summary: Single transaction type search
                value:
                  insuredDetails:
                    - insured:
                        type: Primary
                        entityName: "CocaCola Inc"
                        entityType: Corporation
                        stateOfDomicile: IL
                        dnbNumber: "123456789"
                        countryCode: "US"
                        identifiers:
                          - type: DUNS
                            value: "123456789"
                        aliases: ["GTC Logistics"]
                        addresses:
                          - type: Primary
                            addressLine1: "100 Main Street"
                            city: "Chicago"
                            region: "IL"
                            state: "IL"
                            zipCode: "60061"
                            country: "US"
                            county: "Cook"
                        industryCode: "111120"
                  coverageDetails:
                    lineOfBusiness: ESCasualty
                    classOfBusiness: ["ESCasualty"]
                  searchCriteria:
                    - transactionType: Policy
                      confidenceScore: 50
                      searchPeriodMonths: 6
                      numberOfResults: 10
                      statusFilter: ["Bound", "Cancelled"]
              multipleTransactionTypes:
                summary: Multiple transaction types with different settings
                value:
                  insuredDetails:
                    - insured:
                        type: Primary
                        entityName: "Acme Corp"
                        entityType: Corporation
                        stateOfDomicile: NJ
                        dnbNumber: "555666777"
                        countryCode: "GB"
                        identifiers:
                          - type: DUNS
                            value: "555666777"
                          - type: LEI
                            value: "5493001KJTIIGC8Y1R12"
                  coverageDetails:
                    lineOfBusiness: MarineHull
                    classOfBusiness: ["MarineHull"]
                  searchCriteria:
                    - transactionType: Policy
                      confidenceScore: 50
                      searchPeriodMonths: 6
                      numberOfResults: 10
                      statusFilter: ["Bound", "Cancelled"]
                    - transactionType: Quote
                      confidenceScore: 60
                      searchPeriodMonths: 12
                      numberOfResults: 5
                      statusFilter: ["Quoted", "Referred"]
                    - transactionType: Submission
                      confidenceScore: 40
                      searchPeriodMonths: 3
                      numberOfResults: 20
                      statusFilter: ["Received", "Referred", "Declined"]
      responses:
        '200':
          description: Risk clearance results
          content:
            application/json:
              schema:
                # RiskClearanceResponse (inline)
                type: object
                properties:
                  correlationId:
                    type: string
                    description: Echoed from X-CorrelationId header for tracing.
                    example: "RC-20260317-000123"
                  responseTimestamp:
                    type: string
                    format: date-time
                    example: "2026-03-17T12:45:02Z"
                  provider:
                    type: string
                    description: The provider that returned these results.
                    example: "GSIDataPlatform"
                  status:
                    type: string
                    enum: [Success, Failed, PartialSuccess]
                    example: Success
                  errorCode:
                    type: string
                    description: Error code if provider failed (only present when status is Failed).
                    nullable: true
                    example: null
                  errorMessage:
                    type: string
                    description: Human-readable error message (only present when status is Failed).
                    nullable: true
                    example: null
                  matches:
                    type: array
                    items:
                      type: object  # RiskMatch
                      properties:
                        matchId:
                          type: string
                          description: Unique identifier for this match within the provider.
                          example: "GSI-M-00001"
                        transactionType:
                          type: string
                          enum: [Policy, Quote, Submission, Endorsement, Renewal, Claim]
                          example: Policy
                        confidenceScore:
                          type: number
                          format: double
                          description: Match confidence score as computed by the provider (0.0 - 100.0).
                          example: 85.5
                        status:
                          type: string
                          enum: [Bound, Cancelled, Quoted, Referred, Declined, NotTakenUp, Expired, InForce, Received, Cleared, NotCleared, NotReviewed]
                          example: Bound
                        transactionID: { type: integer, example: 165094 }
                        lineOfBusiness:      { type: string, example: ESCasualty }
                        subLineOfBusiness:   { type: string, example: "LiquorLiability" }
                        policyType:          { type: string, example: Primary }
                        effectiveDate:
                          type: string
                          format: date
                          description: Effective date of the policy/quote/submission.
                          example: "2025-09-15"
                        expirationDate:
                          type: string
                          format: date
                          example: "2026-09-15"
                        inceptionDate:
                          type: string
                          format: date-time
                          example: "2025-09-15T05:16:20Z"
                        policyReference:
                          type: string
                          description: Policy reference number (if applicable).
                          nullable: true
                          example: "POL-2025-44821"
                        quoteReference:
                          type: string
                          description: Quote reference number (if applicable).
                          nullable: true
                          example: null
                        submissionReference:
                          type: string
                          description: Submission reference number (if applicable).
                          nullable: true
                          example: null
                        externalReferenceId: { type: string, example: "70168" }
                        externalReference:   { type: string, example: "QGL101792725" }
                        totalPremium:
                          type: number
                          format: double
                          example: 100.0
                        annualPremium:
                          type: number
                          format: double
                          description: Annualized premium for the policy term.
                          example: 100.0
                        currency: { type: string, example: "USD" }
                        parties:
                          type: array
                          items:
                            type: object  # Party
                            properties:
                              id: { type: string, example: "08DD8BB57460DB16" }
                              insured:
                                type: object  # Insured
                                properties:
                                  type:
                                    type: string
                                    enum: [Primary, Secondary, Additional]
                                    example: Primary
                                  entityName: { type: string }
                                  entityType:
                                    type: string
                                    description: Legal form of insured entity.
                                    example: Corporation
                                  stateOfDomicile: { type: string, example: IL }
                                  dnbNumber: { type: string, example: "123456789" }
                                  addresses:
                                    type: array
                                    items:
                                      type: object  # Address
                                      properties:
                                        type:
                                          type: string
                                          enum: [Primary, Secondary, Billing, Mailing]
                                          example: Primary
                                        addressLine1: { type: string }
                                        addressLine2: { type: string }
                                        city:         { type: string }
                                        region:       { type: string }
                                        state:        { type: string }
                                        zipCode:      { type: string }
                                        country:      { type: string }
                                        county:       { type: string }
                                      additionalProperties: false
                                  industryCode: { type: string, example: "111120" }
                                  webAddress: { type: string }
                                additionalProperties: false
                              broker:
                                type: object  # Broker
                                properties:
                                  type:
                                    type: string
                                    enum: [Primary, Secondary, Additional]
                                    example: Primary
                                  entityName: { type: string }
                                  code: { type: string }
                                  nandaId:
                                    type: string
                                    description: Broker NANDA ID.
                                    example: "NANDA-556677"
                                  contact:
                                    type: object
                                    properties:
                                      firstName:
                                        type: string
                                        description: First name of the contact
                                        example: John
                                      lastName:
                                        type: string
                                        description: Last name of the contact
                                        example: Doe
                                      email: { type: string }
                                    additionalProperties: false
                                  addresses:
                                    type: array
                                    items:
                                      type: object  # Address
                                      properties:
                                        type:
                                          type: string
                                          enum: [Primary, Secondary, Billing, Mailing]
                                          example: Primary
                                        addressLine1: { type: string }
                                        addressLine2: { type: string }
                                        city:         { type: string }
                                        region:       { type: string }
                                        state:        { type: string }
                                        zipCode:      { type: string }
                                        country:      { type: string }
                                        county:       { type: string }
                                      additionalProperties: false
                                additionalProperties: false
                            additionalProperties: false
                        underwriterName:  { type: string, example: "Dawn Fertig" }
                        underwriterEmail: { type: string, example: "johnsmith@munichre.com" }
                        underwriterNid:
                          type: string
                          description: User NID of the underwriter.
                          example: "NID-DF1234"
                        linesOfBusiness:
                          type: array
                          items:
                            type: object  # LineOfBusiness
                            properties:
                              id:        { type: string, example: "14A844F942D3544339EC4767E2BDEFF7F" }
                              type:      { type: string, example: ESCasualty }
                              name:      { type: string, example: ESCasualty }
                              classType: { type: string, example: ESCasualty }
                              exposure:
                                type: object
                                properties:
                                  value: { type: string, example: CGL }
                              limits:
                                type: array
                                items:
                                  type: object
                                  properties:
                                    id:    { type: string }
                                    type:  { type: string, example: ProductsAggregate }
                                    value: { type: number, example: 2000000 }
                                  additionalProperties: false
                              coverages:
                                type: array
                                items:
                                  type: object  # Coverage
                                  properties:
                                    id:   { type: string }
                                    type: { type: string, example: PremisesOperations }
                                    limit:
                                      oneOf:
                                        - type: number
                                        - type: array
                                          items:
                                            type: object
                                            properties:
                                              type: { type: string, example: "17000" }
                                    exposures:
                                      type: array
                                      items:
                                        type: object  # Exposure
                                        properties:
                                          type: { type: string, example: UnitsDivider }
                                          value:
                                            oneOf:
                                              - type: string
                                              - type: number
                                            example: 17000
                                        additionalProperties: false
                                    premium:  { type: number, example: 150 }
                                    selected: { type: boolean, example: true }
                                    deductibles:
                                      type: array
                                      description: Deductibles at the coverage level.
                                      items:
                                        type: object  # Deductible
                                        properties:
                                          type: { type: string, example: PropertyDeductible }
                                          value:
                                            oneOf:
                                              - type: number
                                                format: double
                                              - type: string
                                            example: 500
                                    subCoverages:
                                      type: array
                                      description: Nested coverages with the same structure (recursive).
                                      items: { type: object }
                                  additionalProperties: false
                              deductibles:
                                type: array
                                description: Deductibles at the LoB level.
                                items:
                                  type: object  # Deductible
                                  properties:
                                    type: { type: string, example: PropertyDeductible }
                                    value:
                                      oneOf:
                                        - type: number
                                          format: double
                                        - type: string
                                      example: 2500
                            additionalProperties: false
                        risks:
                          type: array
                          items:
                            type: object  # Risk
                            properties:
                              id: { type: string, example: "r09CB9BA0CAF54BDD80A293E0D9D11B08" }
                              location:
                                type: object
                                properties:
                                  id: { type: string, example: "l17E737137F73493EAB30B807702794C0" }
                                  address:
                                    type: object
                                    properties:
                                      id: { type: string, example: "a30F23DF9F83F49EEBF100ED25CA42E1A" }
                              data:
                                description: Opaque risk-specific blob
                                type: object
                                additionalProperties: true
                              exposures:
                                type: array
                                items:
                                  type: object  # Exposure
                                  properties:
                                    type: { type: string, example: UnitsDivider }
                                    value:
                                      oneOf:
                                        - type: string
                                        - type: number
                                      example: 17000
                                  additionalProperties: false
                              coverages:
                                type: array
                                items:
                                  type: object
                                  allOf:
                                    - type: object  # Coverage (reuse)
                              deductibles:
                                type: array
                                description: Deductibles at the LoB level.
                                items:
                                  type: object  # Deductible
                                  properties:
                                    type: { type: string, example: PropertyDeductible }
                                    value:
                                      oneOf:
                                        - type: number
                                          format: double
                                        - type: string
                                      example: 2500
                            additionalProperties: false
                        underlyingProducts:
                          type: array
                          description: List of underlying policies/products associated with this match.
                          items:
                            type: object
                            properties:
                              id:
                                type: string
                                description: Unique identifier for the underlying product/policy.
                                example: "UD07FE84583A648BDA83094F56E865208D"
                              type:
                                type: string
                                description: Product line/type.
                                example: GeneralLiability
                              name:
                                type: string
                                description: Display name of the underlying product/policy.
                                example: "GeneralLiability"
                              policyComments:
                                type: string
                                description: Free-text comments about the policy.
                                example: "test xlst GL1"
                              carrier:
                                type: string
                                description: Carrier providing the underlying policy.
                                example: "test xlst GL1"
                              writingCompany:
                                type: string
                                description: Writing company name.
                                example: "test xlst GL1"
                              policyNumber:
                                type: string
                                description: Policy number assigned by the carrier.
                                example: "GL-123456"
                              effectiveDate:
                                type: string
                                format: date
                                description: Policy effective date (YYYY-MM-DD).
                                example: "2025-09-01"
                              expirationDate:
                                type: string
                                format: date
                                description: Policy expiration date (YYYY-MM-DD).
                                example: "2025-09-30"
                              coverageVersion:
                                type: string
                                description: Coverage form/version.
                                example: "Claims-Made"
                              retentionStructure:
                                type: string
                                description: Retention structure type.
                                example: "SIR"
                              retentionAmount:
                                type: number
                                format: double
                                minimum: 0
                                description: Retention amount applicable to the policy.
                                example: 10000
                              limitTreatment:
                                type: string
                                description: Whether coverage limit applies inside/outside aggregates.
                                example: "InsideOutside"
                              premium:
                                type: number
                                format: double
                                minimum: 0
                                description: Premium on the underlying policy.
                                example: 100000
                              exposure:
                                type: number
                                format: double
                                minimum: 0
                                description: Exposure value used for rating (e.g., sales, payroll).
                                example: 10000
                              exposureBase:
                                type: string
                                description: Basis of exposure used for rating.
                                example: "GrossSales"
                              retroDate:
                                type: string
                                format: date
                                nullable: true
                                description: Retroactive date for claims-made coverage (if applicable).
                                example: "2020-01-01"
                            additionalProperties: false
                        data:
                          type: object
                          description: Dynamic match-level node for arbitrary provider-specific data.
                          additionalProperties: true
                      additionalProperties: false
                additionalProperties: false
              examples:
                successResponse:
                  summary: Successful response with matches
                  value:
                    correlationId: "RC-20260317-000123"
                    responseTimestamp: "2026-03-17T12:45:02Z"
                    provider: "GSIDataPlatform"
                    status: Success
                    matches:
                      - matchId: "GSI-M-00001"
                        transactionType: Policy
                        confidenceScore: 85.5
                        status: Bound
                        transactionID: 165094
                        lineOfBusiness: ESCasualty
                        subLineOfBusiness: "LiquorLiability"
                        policyType: Primary
                        effectiveDate: "2025-09-15"
                        expirationDate: "2026-09-15"
                        inceptionDate: "2025-09-15T05:16:20Z"
                        policyReference: "POL-2025-44821"
                        externalReferenceId: "70168"
                        externalReference: "QGL101792725"
                        totalPremium: 100.0
                        annualPremium: 100.0
                        currency: "USD"
                        parties:
                          - id: "08DD8BB57460DB16"
                            insured:
                              type: Primary
                              entityName: "DNT_Demo_SC01"
                              entityType: Corporation
                              stateOfDomicile: IL
                              dnbNumber: "123456789"
                              addresses:
                                - type: Primary
                                  addressLine1: "100 Main Street"
                                  addressLine2: "Apt 101"
                                  city: "Chicago"
                                  region: "IL"
                                  state: "IL"
                                  zipCode: "60061"
                                  country: "US"
                                  county: "Cook"
                              webAddress: "https://google.com"
                            broker:
                              type: Primary
                              entityName: "Jimcor Agency, Inc. T/A Jimcor Agencies"
                              nandaId: "NANDA-556677"
                              addresses:
                                - type: Primary
                                  addressLine1: "60 Craig Road"
                                  addressLine2: "Suite 200"
                                  city: "Montville"
                                  region: "NJ"
                                  zipCode: "07645"
                              contact:
                                firstName: "John"
                                lastName: "Smith"
                                email: "John@aol.com"
                        underwriterName: "Dawn Fertig"
                        underwriterNid: "NID-DF1234"
                        underwriterEmail: "johnsmith@munichre.com"
                        linesOfBusiness:
                          - id: "14A844F942D3544339EC4767E2BDEFF7F"
                            type: ESCasualty
                            name: ESCasualty
                            classType: ESCasualty
                            exposure: { value: CGL }
                            limits:
                              - id: "1C0FC7014B2E14AA78748273647528C21"
                                type: ProductsAggregate
                                value: 2000000
                            coverages:
                              - id: "c3502C534EA76494AA1B746B459D6DBB7"
                                type: PremisesOperations
                                limit:
                                  - { type: "17000" }
                                premium: 150
                                deductibles:
                                  - type: PropertyDeductible
                                    value: 500
                            deductibles:
                              - type: PropertyDeductible
                                value: 2500
                        risks:
                          - id: "r09CB9BA0CAF54BDD80A293E0D9D11B08"
                            location:
                              id: "l17E737137F73493EAB30B807702794C0"
                              address: { id: "a30F23DF9F83F49EEBF100ED25CA42E1A" }
                            data: {}
                            exposures:
                              - { type: UnitsDivider, value: "10010" }
                              - { value: "17000" }
                        underlyingProducts:
                          - id: "UD07FE84583A648BDA83094F56E865208D"
                            type: GeneralLiability
                            name: "GeneralLiability"
                            policyComments: "test xlst GL1"
                            carrier: "test xlst GL1"
                            writingCompany: "test xlst GL1"
                            policyNumber: "GL-123456"
                            effectiveDate: "2025-09-01"
                            expirationDate: "2025-09-30"
                            coverageVersion: "Claims-Made"
                            retentionStructure: "SIR"
                            retentionAmount: 10000
                            limitTreatment: "InsideOutside"
                            premium: 100000
                            exposure: 10000
                            exposureBase: "GrossSales"
                            retroDate: "2020-01-01"
                        data:
                          deepLink: "http://ued39.duckcreekondemand.com/Policy/policies/10264/modelCollections/MainInsured"
                          expiringOptionId: 1434
                providerFailure:
                  summary: Provider failure response
                  value:
                    correlationId: "RC-20260317-000456"
                    responseTimestamp: "2026-03-17T14:30:05Z"
                    provider: "LloydsMDR"
                    status: Failed
                    errorCode: "PROVIDER_TIMEOUT"
                    errorMessage: "LloydsMDR did not respond within 30 seconds"
                    matches: []
        '400':
          description: Invalid request payload
          content:
            application/json:
              schema:
                type: object
                properties:
                  correlationId: { type: string }
                  errorCode: { type: string, enum: [VALIDATION_ERROR] }
                  errors:
                    type: array
                    items:
                      type: object
                      properties:
                        field: { type: string }
                        message: { type: string }
              example:
                correlationId: "RC-20260317-000123"
                errorCode: "VALIDATION_ERROR"
                errors:
                  - field: "insuredDetails"
                    message: "At least one insured detail is required"
                  - field: "searchCriteria"
                    message: "At least one search criteria entry is required"
        '401':
          description: Missing or invalid JWT token
        '429':
          description: Rate limit exceeded
        '502':
          description: Provider returned an error
        '504':
          description: Provider did not respond within configured timeout
```
