# Poetic Commerce Protocol (PCP)

## 1. Introduction

The Poetic Commerce Protocol (PCP) is a standardized, machine-readable protocol designed to enable **AI agents** to seamlessly discover, understand, and transact with online merchants. It eliminates the need for custom API integrations by providing a universal layer for agentic commerce.

## 2. Core Concepts

*   **Agent:** An AI program acting on behalf of a user for commercial activities (e.g., product discovery, purchasing).
*   **Merchant:** An entity selling goods or services, represented by a `schema:Organization`.
*   **Manifest:** A JSON-LD document hosted by the Merchant. It's the entry point for an Agent, declaring the merchant's identity, capabilities, API endpoints, and authentication requirements.
*   **PCP Vocabulary:** An RDF vocabulary extending Schema.org with commerce-specific concepts like `poetic:inventoryId` and `poetic:authentication`.

## 3. How PCP Works (Protocol Flow)

PCP operates on a simple discovery-and-interaction model:

1.  **Discovery:** An Agent finds the Merchant's Manifest file.
2.  **Understanding:** The Agent parses the Manifest to identify API endpoints (catalog, checkout, order status) and authentication methods.
3.  **Authentication:** The Agent authenticates with the Merchant's system using credentials obtained from the Manifest.
4.  **Interaction:** The Agent uses authenticated credentials to interact with standardized API endpoints for tasks like fetching products or placing orders.

### 3.1. Manifest Discovery

Merchants should make their Manifest discoverable via:

1.  **Well-Known URI:** `/.well-known/poetic-manifest.json` relative to their domain root.
2.  **HTML Link Tag:** `<link rel="poetic-manifest" href="https://path/to/your/manifest.json">` in their website's `<head>`.

## 4. The Merchant Manifest

The Manifest is a **REQUIRED** JSON-LD document, serving as the cornerstone of PCP. Its root object **MUST** be `@type: schema:Organization`.

### 4.1. Key Properties

| Property                 | Type                               | Requirement  | Description                                                                                                                                                           |
| ------------------------ | ---------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@context`               | Object                             | **REQUIRED** | JSON-LD context, including `"schema": "https://schema.org/"` and `"poetic": "https://specs.poetic.com/v1/"`.                                                 |
| `@type`                  | String                             | **REQUIRED** | Must be `schema:Organization`.                                                                                                                                        |
| `@id`                    | URL (String)                       | **REQUIRED** | A stable, unique URI identifying the organization.                                                                                           |
| `name`                   | String                             | **REQUIRED** | The organization's legal name.                                                                                                                                   |
| `url`                    | URL (String)                       | **RECOMMENDED**| Primary URL of the merchant's website.                                                                                                             |
| `logo`                   | URL (String)                       | **RECOMMENDED**| URL to the organization's logo.                                                                                                                                     |
| `poetic:protocolVersion` | String                             | **REQUIRED** | The PCP specification version this manifest conforms to (e.g., `"1.0"`).                                                          |
| `poetic:catalogEndpoint` | URL (String)                       | **REQUIRED** | Absolute URL for the Product Catalog API endpoint.                                                                                                 |
| `poetic:authentication`  | `poetic:Authentication` Object     | **REQUIRED** | Details supported authentication methods. See Section 5.                                        |
| `potentialAction`        | `schema:BuyAction` Object          | **REQUIRED** | Defines the checkout process entry point, with a critical `target` property.                                                                |
| `poetic:orderStatusEndpoint` | `schema:EntryPoint` Object         | **RECOMMENDED**| An EntryPoint with a `urlTemplate` (e.g., `{orderId}`) for checking order status. |

## 5. Authentication

PCP requires secure communication. The `poetic:authentication` object specifies the mechanism.

### 5.1. `poetic:Authentication` Object

*   `@type`: MUST be `poetic:Authentication`.
*   `supportedMethods`: An array of `poetic:AuthMethod` objects. Agents should use the first supported method.

### 5.2. `poetic:AuthMethod` Object

*   `@type`: MUST be `poetic:AuthMethod`.
*   `method`: A string enum (e.g., `"OAuth2ClientCredentials"`).
*   `tokenUrl`: **REQUIRED** for OAuth. The authorization server's token endpoint URL.
*   `requiredScope`: **RECOMMENDED** for OAuth. Space-delimited scope values (e.g., `"commerce.all"`).

### 5.3. Example Flow: OAuth 2.0 Client Credentials

1.  Agent parses `poetic:authentication` from Manifest.
2.  Agent makes `POST` request to `tokenUrl` with `Content-Type: application/x-www-form-urlencoded` and `grant_type=client_credentials&client_id=YOUR_CLIENT_ID&client_secret=YOUR_CLIENT_SECRET`.
3.  Authorization server returns JSON with `access_token`.
4.  Agent includes `Authorization: Bearer <access_token>` header in all subsequent requests to PCP endpoints.

## 6. Core Workflows and API Endpoints

All API responses **SHOULD** be `application/ld+json`.

### 6.1. Fetching the Product Catalog

*   **Objective:** Discover merchant products.
*   **Endpoint:** `poetic:catalogEndpoint` from Manifest.
*   **HTTP Request:** `GET` with `Accept: application/ld+json` and `Authorization: Bearer <token>`.
*   **Success Response (200 OK):** JSON-LD document with a list of `schema:Offer` objects. Each `schema:Offer` **MUST** include `poetic:inventoryId` (immutable transactional identifier).

### 6.2. Placing an Order (Checkout)

*   **Objective:** Purchase items.
*   **Endpoint:** `urlTemplate` from `potentialAction` (`@type: schema:BuyAction`) in Manifest.
*   **HTTP Request:** `POST` (or `httpMethod` specified) with `Content-Type: application/ld+json` and `Authorization: Bearer <token>`.
*   **Request Body:** A `schema:Order` object. `orderedItem` **MUST** be an array of `schema:OrderItem` objects. Each `schema:OrderItem`'s `orderedItem` **MUST** be a minimal `schema:Offer` with `@type` and `poetic:inventoryId`.
*   **Success Response (201 Created or 200 OK):** A `schema:Order` object with `orderNumber` (merchant-specific ID) and `orderStatus` (e.g., `schema:OrderProcessing`).

### 6.3. Checking Order Status

*   **Objective:** Get latest order status.
*   **Endpoint:** `urlTemplate` from `poetic:orderStatusEndpoint` in Manifest.
*   **HTTP Request:** `GET` (or `httpMethod` specified) with `Accept: application/ld+json` and `Authorization: Bearer <token>`. Replace `{orderId}` in `urlTemplate` with `orderNumber`.
*   **Success Response (200 OK):** A `schema:Order` object with complete, up-to-date order details, including `orderStatus`.

## 7. PCP v1.0 Vocabulary Reference

This section defines terms in the `poetic` namespace.

### 7.1. Classes

*   `poetic:Authentication`: Describes an authentication scheme for Poetic endpoints.
*   `poetic:AuthMethod`: Subclass of `schema:Intangible` describing a specific authentication method (e.g., OAuth 2.0 Client Credentials).

### 7.2. Properties

*   `poetic:protocolVersion`
    *   Domain: `schema:Organization`
    *   Range: `schema:Text`
    *   Comment: Version of the PCP specification this manifest adheres to.
*   `poetic:catalogEndpoint`
    *   Domain: `schema:Organization`
    *   Range: `schema:URL`
    *   Comment: URL for retrieving the merchant's product catalog.
*   `poetic:orderStatusEndpoint`
    *   Domain: `schema:Organization`
    *   Range: `schema:EntryPoint`
    *   Comment: EntryPoint URL template for checking order status.
*   `poetic:authentication`
    *   Domain: `schema:Organization`
    *   Range: `poetic:Authentication`
    *   Comment: Descriptor object specifying authentication methods for Poetic endpoints.
*   `poetic:inventoryId`
    *   Domain: `schema:Offer`
    *   Range: `schema:Text`
    *   Comment: Unique, immutable identifier for a specific offer, used for transactional integrity.
