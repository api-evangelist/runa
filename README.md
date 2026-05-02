# Runa

Runa is a global digital payouts platform that enables businesses to automate digital reward and gift card distribution through a single API. The platform provides access to over 5,000 gift cards and payout options across 190+ countries, supporting B2C payments, employee rewards, customer incentives, and loyalty programs.

**Human URL:** https://developer.runa.io/

## APIs

| API | Description |
|-----|-------------|
| [Runa Payouts API](openapi/runa-payouts-api-openapi.yml) | Digital reward and gift card distribution via orders, products, and balance management |

## OpenAPI Specifications

| Spec | Description |
|------|-------------|
| [runa-payouts-api-openapi.yml](openapi/runa-payouts-api-openapi.yml) | Payouts API covering orders, products, categories, countries, balance, and ping |

## Rules

| Ruleset | Description |
|---------|-------------|
| [runa-spectral-rules.yml](rules/runa-spectral-rules.yml) | Spectral ruleset enforcing Runa API design conventions |

## Capabilities

### Workflow Capabilities

| Capability | Description |
|------------|-------------|
| [rewards-distribution.yaml](capabilities/rewards-distribution.yaml) | End-to-end digital reward distribution workflow |

### Shared Definitions

| Shared | Description |
|--------|-------------|
| [payouts-api.yaml](capabilities/shared/payouts-api.yaml) | Per-API consumed definition for the Runa Payouts API |

## Schemas

| Schema | Description |
|--------|-------------|
| [runa-order-schema.json](json-schema/runa-order-schema.json) | JSON Schema for Runa order records |

## Structures

| Structure | Description |
|-----------|-------------|
| [runa-structure.json](json-structure/runa-structure.json) | JSON structure documentation for Runa data entities |

## JSON-LD

| Context | Description |
|---------|-------------|
| [runa-context.jsonld](json-ld/runa-context.jsonld) | JSON-LD context mapping Runa vocabulary to schema.org |

## Examples

| Example | Description |
|---------|-------------|
| [runa-create-order-example.json](examples/runa-create-order-example.json) | Create a $25 multi-brand payout link order |
| [runa-get-balance-example.json](examples/runa-get-balance-example.json) | Get USD account balance |

## Vocabulary

| Vocabulary | Description |
|------------|-------------|
| [runa-vocabulary.yml](vocabulary/runa-vocabulary.yml) | Domain vocabulary for digital rewards, payouts, and gift card concepts |

## Maintainers

**FN:** Kin Lane  
**Email:** kin@apievangelist.com
