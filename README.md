Practice 01 — Message Routing & Transformation

Course
Enterprise Application Integration

Objective


Build an integration flow that:

-Accepts orders from 3 different source systems

-Routes them using a Content-Based Router

-Transforms them into a Canonical Order Model

-Enriches order items using a live Pricing API

-Returns the enriched canonical order downstream

Patterns used (from Enterprise Integration Patterns):

-Message Translator

-Content-Based Router

-Content Enricher

-Pipes & Filters

-Canonical Data Model



Architecture Overview

Integration Flow:

[HTTP IN]
      ↓
[Content-Based Router]
      ↓
[Translator per Source]
      ↓
[Product ID Normalization]
      ↓
[Pricing API Call]
      ↓
[Attach Pricing]
      ↓
[Content Filter]
      ↓
[HTTP Response]


Source Systems
1️ Web Store

Format: JSON (Nested)

Example:

{
  "orderId": "WEB-123",
  "customerEmail": "janis@ex.com",
  "items": [
    {
      "productCode": "PROD-001",
      "quantity": 1,
      "priceEur": 899
    }
  ],
  "orderDate": "2026-02-17T10:30Z",
  "shippingMethod": "express"
}
2️ Mobile App

Format: JSON (Flat)

3️ B2B Partner

Format: XML

Requires:

-XML → JSON conversion

-Product ID normalization

-Grouping multiple rows into single order

Canonical Order Model

All orders are transformed into the following JSON structure:

{
  "schemaVersion": "1.0",
  "canonicalOrderId": "string",
  "sourceSystem": "WEB | MOBILE | B2B",
  "receivedAt": "ISO-8601",

  "customer": {
    "name": "string",
    "email": "string",
    "phone": "string",
    "company": "string",
    "externalReferences": []
  },

  "items": [
    {
      "productId": "PROD-XXX",
      "quantity": 0,
      "unitPrice": 0.0,
      "currency": "ISO-4217",
      "taxRate": 0.0
    }
  ],

  "shipping": {
    "method": "express | standard | bulk"
  },

  "timestamps": {
    "orderCreated": "ISO-8601"
  }
}
Design Principles

-Superset model (supports all systems)

-ISO-8601 timestamps

-ISO-4217 currency codes

-Optional fields allowed

-Self-describing (sourceSystem + schemaVersion)

Field Mapping
Web → Canonical
Web Field	Canonical Field
orderId	canonicalOrderId
customerEmail	customer.email
productCode	items.productId
quantity	items.quantity
priceEur	items.unitPrice
orderDate	timestamps.orderCreated
shippingMethod	shipping.method

Mobile → Canonical

Mobile Field	Canonical
oid	canonicalOrderId
email	customer.email
sku	items.productId
ts	timestamps.orderCreated

Product ID normalized (see below).

B2B XML → Canonical

XML	Canonical
ref	canonicalOrderId
cust_name	customer.name
cust_phone	customer.phone
sku	items.productId
qty	items.quantity
unit_price	items.unitPrice
currency	items.currency
order_ts	timestamps.orderCreated
ship	shipping.method

Date converted to ISO-8601 format.

Product ID Normalization

Required before calling Pricing API.

Source	Raw ID	Normalized
Web	PROD-001	PROD-001
Mobile	001	PROD-001
B2B	SKU-PROD-001	PROD-001

Logic:

-Add prefix PROD- if missing

-Remove prefix SKU- if present

Content Enrichment — Pricing API

Each item is enriched using:

GET http://pricing-api:3000/pricing/:id

Data appended to each item:

-unitPrice

-currency

-taxRate

Prices are NOT hardcoded.

Content Filtering

Before sending response downstream:
Removed fields:

-customer.phone
-customer.email (if external integration)

Allowlist approach preferred for external systems.

Schema Evolution

Scenario: Web adds "giftWrap": true
Solution:

-Add optional field giftWrap to canonical model
-Do NOT make mandatory
-Other systems ignore it

Backward compatible change.

Deployment Instructions

1️Start Docker

In project folder:

docker compose up --build

Wait until containers start.

2️ Open Node-RED

http://localhost:1880

3️ Test Endpoint

POST:

http://localhost:1880/order

Example body:

{
  "sourceSystem": "WEB",
  "orderId": "WEB-123",
  "customerEmail": "test@test.com",
  "items": [
    {
      "productCode": "001",
      "quantity": 1,
      "priceEur": 100
    }
  ],
  "orderDate": "2026-02-17T10:30Z",
  "shippingMethod": "express"
}

Architectural Benefits

Without Canonical Model:

N × (N−1) translators

With Canonical Model:

2 × N translators

Scales significantly better as systems grow.

Conclusion

This solution demonstrates:

-Proper use of EAI transformation patterns
-Clean modular flow (Pipes & Filters)
-Canonical Data Model design
-Runtime enrichment
-Schema evolution awareness

The integration flow is fully functional and compliant with lab requirements.
