# Poetic Commerce Protocol (PCP) Specification
**Version:** 1.0
**Status:** Stable

### 1. Overview

This document provides the normative specification for the Poetic Commerce Protocol (PCP). PCP is a decentralized protocol designed to facilitate **agentic commerce**, enabling AI agents to discover, understand, and transact with online merchants in a standardized way.

The protocol's primary goal is to create a universal, machine-readable layer for commerce, abstracting away the need for bespoke API integrations for each merchant. It achieves this by leveraging open standards, including JSON-LD, Schema.org, and RDF.

#### 1 Core Concepts

*   **Agent:** An autonomous or semi-autonomous software program that acts on behalf of a user to perform commercial activities (e.g., product discovery, price comparison, purchasing).
*   **Merchant:** An entity offering goods or services for sale. In PCP, a merchant is represented by a `schema:Organization`.
*   **Manifest:** A JSON-LD document, hosted by the Merchant, that serves as the entry point for an Agent. It declares the merchant's identity, capabilities, API endpoints, and authentication requirements.
*   **PCP Vocabulary:** A formal RDF vocabulary (ontology) that extends Schema.org with concepts specific to agentic commerce, such as `poetic:inventoryId` and `poetic:authentication`.

---

### 2. Protocol Architecture

The protocol operates on a discovery-and-interaction model.

1.  **Discovery:** The Agent discovers the Merchant's Manifest file.
2.  **Parsing & Understanding:** The Agent parses the JSON-LD Manifest to identify key API endpoints (`catalog`, `checkout`, `orderStatus`) and understand the required authentication scheme.
3.  **Authentication:** The Agent authenticates with the Merchant's system according to the `poetic:authentication` block in the Manifest, obtaining an access token or other credentials.
4.  **Interaction:** The Agent uses the authenticated credentials to interact with the Merchant's standardized API endpoints to perform actions like fetching the product catalog, placing an order, and checking order status.

#### 2.1. Manifest Discovery

Merchants SHOULD make their Manifest discoverable via one of the following methods:

1.  **Well-Known URI:** By placing the manifest at the following path relative to their domain root:
    `/.well-known/poetic-manifest.json`
2.  **HTML Link Tag:** By including a `<link>` tag in the `<head>` of their main website's HTML:
    `<link rel="poetic-manifest" href="https://path/to/your/manifest.json">`

---

### 3. The Merchant Manifest Specification

The Manifest is the cornerstone of the protocol. It MUST be a valid JSON-LD document.

#### 3.1. Root Object

The root object of the Manifest MUST be of `@type: schema:Organization`.

#### 3.2. Properties

| Property                 | Type                               | Requirement  | Description                                                                                                                                                           |
| ------------------------ | ---------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@context`               | Object                             | **REQUIRED** | The JSON-LD context. MUST include `"schema": "https://schema.org/"` and `"poetic": "https://specs.poetic.com/v1/"`.                                                 |
| `@type`                  | String                             | **REQUIRED** | MUST be `schema:Organization`.                                                                                                                                        |
| `@id`                    | URL (String)                       | **REQUIRED** | A stable, unique, and dereferenceable URI that identifies the organization.                                                                                           |
| `name`                   | String                             | **REQUIRED** | The legal name of the organization.                                                                                                                                   |
| `url`                    | URL (String)                       | **RECOMMENDED**| The primary URL of the merchant's human-readable website.                                                                                                             |
| `logo`                   | URL (String)                       | **RECOMMENDED**| A URL to the organization's logo.                                                                                                                                     |
| `poetic:protocolVersion` | String                             | **REQUIRED** | The version of the PCP specification this manifest conforms to. For this version, the value MUST be `"1.0"`.                                                          |
| `poetic:catalogEndpoint` | URL (String)                       | **REQUIRED** | The absolute URL for the machine-readable Product Catalog API endpoint.                                                                                                 |
| `poetic:authentication`  | `poetic:Authentication` Object     | **REQUIRED** | An object detailing the supported authentication methods required to access the Poetic API endpoints. See Section 4 for details.                                        |
| `potentialAction`        | `schema:BuyAction` Object          | **REQUIRED** | Defines the entry point for the checkout process. The `target` property within this action is critical.                                                                |
| `poetic:orderStatusEndpoint` | `schema:EntryPoint` Object         | **RECOMMENDED**| An EntryPoint object containing a `urlTemplate` for agents to check the status of a previously placed order. The template SHOULD include a variable like `{orderId}`. |

---

### 4. Authentication

PCP requires secure communication. The `poetic:authentication` object specifies the mechanism.

#### 4.1. `poetic:Authentication` Object Structure

*   `@type`: MUST be `poetic:Authentication`.
*   `supportedMethods`: An array of `poetic:AuthMethod` objects. An Agent SHOULD use the first method in the array that it supports.

#### 4.2. `poetic:AuthMethod` Object Structure

*   `@type`: MUST be `poetic:AuthMethod`.
*   `method`: A string enum identifying the method. The initial spec defines one value:
    *   `"OAuth2ClientCredentials"`: The OAuth 2.0 Client Credentials Grant flow.
*   `tokenUrl`: **REQUIRED** for OAuth. The URL of the authorization server's token endpoint.
*   `requiredScope`: **RECOMMENDED** for OAuth. A space-delimited string of scope values required to access the commerce APIs (e.g., `"commerce.all"`).

#### 4.3. Example Flow: OAuth 2.0 Client Credentials

1.  The Agent parses the `poetic:authentication` block from the Manifest.
2.  The Agent makes a `POST` request to the `tokenUrl`.
    *   **Headers**: `Content-Type: application/x-www-form-urlencoded`
    *   **Body**: `grant_type=client_credentials&client_id=YOUR_CLIENT_ID&client_secret=YOUR_CLIENT_SECRET`
3.  The authorization server validates the credentials and returns a JSON object containing an `access_token`.
4.  For all subsequent requests to PCP endpoints (catalog, checkout, etc.), the Agent MUST include the token in the HTTP `Authorization` header:
    `Authorization: Bearer <access_token>`

---

### 5. Core Workflows and API Endpoints

This section details the standardized interactions an Agent performs. All API responses SHOULD be `application/ld+json`.

#### 5.1. Workflow: Fetching the Product Catalog

*   **Objective:** Discover the products a merchant offers.
*   **Endpoint Source:** The URL from the `poetic:catalogEndpoint` property in the Manifest.
*   **HTTP Request:**
    *   Method: `GET`
    *   Headers: `Accept: application/ld+json`, `Authorization: Bearer <token>`
*   **Success Response (200 OK):**
    *   A JSON-LD document containing a list of objects. Each object MUST be a `schema:Offer`.
    *   Each `schema:Offer` object **MUST** include the `poetic:inventoryId` property. This ID is the immutable, transactional identifier for that specific offer variation (e.g., a specific size and color of a shirt).

#### 5.2. Workflow: Placing an Order (Checkout)

*   **Objective:** Purchase one or more items from the catalog.
*   **Endpoint Source:** The `urlTemplate` from the `target` of the `potentialAction` (`@type: schema:BuyAction`) in the Manifest.
*   **HTTP Request:**
    *   Method: `POST` (or as specified by `httpMethod` in the `target`).
    *   Headers: `Content-Type: application/ld+json`, `Authorization: Bearer <token>`
    *   **Request Body:** A `schema:Order` object.
        *   The `orderedItem` property MUST be an array of `schema:OrderItem` objects.
        *   Each `schema:OrderItem`'s `orderedItem` property MUST be a minimal `schema:Offer` object containing the `@type` and the `poetic:inventoryId` of the item being purchased. This ensures transactional integrity.

*   **Success Response (201 Created or 200 OK):**
    *   A `schema:Order` object representing the newly created order.
    *   This response object MUST include the `orderNumber` (the merchant-specific order ID) and an `orderStatus` (e.g., `schema:OrderProcessing`).

#### 5.3. Workflow: Checking Order Status

*   **Objective:** Get the latest status of a previously placed order.
*   **Endpoint Source:** The `urlTemplate` from `poetic:orderStatusEndpoint` in the Manifest.
*   **HTTP Request:**
    *   The Agent MUST replace the `{orderId}` variable in the `urlTemplate` with the `orderNumber` received from the checkout response.
    *   Method: `GET` (or as specified by `httpMethod`).
    *   Headers: `Accept: application/ld+json`, `Authorization: Bearer <token>`
*   **Success Response (200 OK):**
    *   A `schema:Order` object containing the complete, up-to-date details of the requested order, including the current `orderStatus`.

---

### 6. PCP v1.0 Vocabulary Reference

This section formally defines the terms in the `poetic` namespace.

#### 6.1. Classes

*   `poetic:Authentication`: A class that describes an authentication scheme for Poetic endpoints. It acts as a container for one or more specific authentication methods.
*   `poetic:AuthMethod`: A subclass of `schema:Intangible` that describes a specific method of authentication, such as OAuth 2.0 Client Credentials.

#### 6.2. Properties

*   `poetic:protocolVersion`
    *   Domain: `schema:Organization`
    *   Range: `schema:Text`
    *   Comment: The version of the Poetic Commerce Protocol that this manifest adheres to.
*   `poetic:catalogEndpoint`
    *   Domain: `schema:Organization`
    *   Range: `schema:URL`
    *   Comment: The fully qualified URL endpoint for retrieving the merchant's machine-readable product catalog.
*   `poetic:orderStatusEndpoint`
    *   Domain: `schema:Organization`
    *   Range: `schema:EntryPoint`
    *   Comment: An EntryPoint URL template that an agent can use to check the status of a previously placed order.
*   `poetic:authentication`
    *   Domain: `schema:Organization`
    *   Range: `poetic:Authentication`
    *   Comment: A descriptor object that specifies the authentication methods required to interact with the Poetic endpoints.
*   `poetic:inventoryId`
    *   Domain: `schema:Offer`
    *   Range: `schema:Text`
    *   Comment: The unique, immutable identifier for a specific offer within the Poetic Unified Data Model, used for transactional integrity. It MUST uniquely identify a purchasable item (e.g., SKU, UPC, or internal ID).
