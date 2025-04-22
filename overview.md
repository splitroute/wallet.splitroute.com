**(Version: 0.1.1)** - *Incorporating Concurrency Control (429) and other improvements.*

**(Generated from Project Structure and Code)**

---

**QUICK START FOR DEVELOPERS**

1.  **Base URL:** Identify where the API is hosted (e.g., `http://localhost:8000`).
2.  **Public Info:** Use `GET /accounts/{nano_address}/...` endpoints to view any account's balance, history, etc. No authentication needed.
3.  **Create Wallet:** Use `POST /wallet/create` endpoint to generate a new Nano wallet with seed, private key, and address. Store these credentials securely.
4.  **Wallet Actions (Send/Receive/Pay):**
    *   Use `POST /wallet/...` and `POST /invoices/pay/...` endpoints.
    *   **CRITICAL:** Authentication requires your **Nano Private Key OR Seed+Index** in HTTP headers (`X-Wallet-Private-Key` **or** `X-Wallet-Seed` + `X-Wallet-Index`). **NEVER expose these in front-end code. Treat them like passwords.** See [Authentication](#authentication-critical-read) section below.
    *   **SEQUENTIAL ACTIONS:** Authenticated operations that modify the wallet state (`send`, `receive_all`, `receive_one`, `pay_invoice`) for a **single account** must be performed **sequentially**. Nano requires blocks to be added one after another. This API enforces this with internal locking. Concurrent requests to modify the same wallet will result in a `429 Too Many Requests` error. Clients should handle this by retrying the request later (see [Error Handling](#error-handling)).
    *   To send, use `POST /wallet/send`. Provide amount as `amount_nano`, `amount_raw`, OR `nominal_amount` + `nominal_currency` (e.g., "EUR", "USD" - this calculates the Nano amount to send based on exchange rates).
    *   To receive pending funds, use `POST /wallet/receive_all`.
    *   To pay a SplitRoute invoice, use `POST /invoices/pay/{invoice_id}`.
5.  **Check Examples:** See [Use Cases & Examples](#use-cases--examples) for `curl` and Python snippets.
6.  **Error Handling:** Check HTTP status codes (4xx = client error, 5xx = server error) and the `detail` field in the JSON response body. Pay special attention to the `429` status code for authenticated operations.

---

## Table of Contents

1.  [Overview](#overview)
    *   [What is this API?](#what-is-this-api)
    *   [Target Audience](#target-audience)
    *   [Key Features](#key-features)
2.  [Getting Started](#getting-started)
    *   [Prerequisites](#prerequisites)
    *   [API Endpoint Base URL](#api-endpoint-base-url)
    *   [**Authentication (CRITICAL READ)**](#authentication-critical-read) <--- **⚠️ READ THIS CAREFULLY! ⚠️**
3.  [Core Concepts](#core-concepts)
    *   [Nano vs. Raw Units](#nano-vs-raw-units)
    *   [Private Keys, Seeds, and Indices](#private-keys-seeds-and-indices)
    *   [Accounts and Blocks](#accounts-and-blocks)
    *   [**Sequential Operations & Concurrency**](#sequential-operations--concurrency) <--- **IMPORTANT**
4.  [API Reference](#api-reference)
    *   [Health Check](#health-check)
    *   [Public Account Information](#public-account-information)
    *   [Authenticated Wallet Operations](#authenticated-wallet-operations) <--- Requires Special Headers & Sequential Execution
    *   [Exchange Rate Information](#exchange-rate-information)
    *   [SplitRoute Invoices](#external-invoices-splitroute)
5.  [Data Types (DTOs)](#data-types-dtos)
6.  [Error Handling](#error-handling) <--- Includes `429` Explanation
7.  [Use Cases & Examples](#use-cases--examples)
8.  [Security Considerations](#security-considerations) <--- Includes Handling `429`
9.  [Audit Logging and Chain Validation](#audit-logging-and-chain-validation)
    *   [Audit Log Features](#audit-log-features)
    *   [Audit Log Format](#audit-log-format)

---

## 1. Overview

### What is this API?

The Nano Wallet API provides a programmatic interface to interact with the Nano cryptocurrency network. It simplifies common tasks like checking balances, viewing history, sending/receiving funds, getting exchange rate information, and interacting with SplitRoute.

### Target Audience

*   **Developers:** Integrating Nano wallet features into applications.
*   **LLM Agents:** Accessing Nano data and performing wallet actions programmatically.
*   **Scripters:** Automating Nano-related tasks.

### Key Features

*   Standard RESTful interface (HTTP/JSON).
*   Public data access for any Nano account.
*   Authenticated operations using Nano keys/seeds.
*   **Built-in locking** to ensure sequential operations on authenticated wallets.
*   Flexible sending (Nano, raw, or calculated from fiat value).
*   Exchange rate lookups (via Kraken).
*   SplitRoute invoice interaction.

---

## 2. Getting Started

### Prerequisites

*   Understanding of basic API concepts (HTTP, JSON).
*   **For Authenticated Operations:** You MUST have the **Private Key** or **Seed + Index** for the Nano account you wish to control.
*   **For Sending/Receiving:** The controlled account needs sufficient balance to send or pending blocks to receive.
*   Understanding that **authenticated actions on a single account must be sequential** (see [Sequential Operations & Concurrency](#sequential-operations--concurrency)).

### API Endpoint Base URL

All requests are made relative to the API's deployment URL (e.g., `http://localhost:8000`, `https://your-nano-api.com`).

### **Authentication (CRITICAL READ)**

> **🚨☠️ EXTREMELY IMPORTANT WARNING ☠️🚨**
>
> *   This API **DOES NOT USE** standard API keys for protected actions.
> *   Authentication for endpoints like `/wallet/*` and `/invoices/pay/*` is done by providing your actual **NANO PRIVATE KEY** or **NANO SEED + INDEX** directly in HTTP headers.
> *   **THIS MEANS THESE HEADERS GIVE FULL, IRREVOCABLE CONTROL OVER YOUR NANO FUNDS.**
> *   **NEVER** put these headers in client-side code (e.g., Browser JavaScript, mobile apps). An attacker could easily steal them.
> *   **ONLY** make authenticated calls from a **SECURE BACKEND ENVIRONMENT** that you fully control and trust.
> *   **TREAT THESE HEADERS WITH THE SAME EXTREME SECURITY AS YOUR NANO KEYS/SEED.** Do not log them, store them insecurely, or transmit them unencrypted.
> *   **USE HTTPS** (TLS/SSL) to encrypt these headers during transmission. Using plain HTTP is **EXTREMELY DANGEROUS**.

**Authentication Methods (Provide EXACTLY ONE set of headers per request):**

1.  **Private Key Method:**
    *   `X-Wallet-Private-Key`: (string) Your 64-character hex Nano private key.

2.  **Seed + Index Method:**
    *   `X-Wallet-Seed`: (string) Your 64-character hex Nano seed.
    *   `X-Wallet-Index`: (string) The account index (e.g., "0", "1") derived from the seed.

**Failure to provide valid headers for protected endpoints results in `401 Unauthorized`. Providing both methods simultaneously is also an error.**

---

## 3. Core Concepts

### Nano vs. Raw Units

*   **Nano (XNO):** Standard decimal unit (e.g., `1.5`). Typically used for user display.
*   **Raw:** Smallest indivisible unit (1 XNO = 10^30 raw). Represented as a large integer *string* (e.g., `"15000000000000000000000000000000"`). The API uses raw units internally and for fields like `balance_raw`, `amount_raw`. Calculations should ideally use raw to avoid floating-point issues.

### Private Keys, Seeds, and Indices

*   **Private Key:** A secret 64-character hexadecimal string that directly controls a *single* Nano account. Compromise means loss of funds for that account.
*   **Seed:** A secret 64-character hexadecimal string that can deterministically generate *multiple* private keys/accounts. Compromise means loss of funds for *all* accounts derived from that seed.
*   **Index:** A non-negative integer (0, 1, 2...) used with a seed to select *which* specific account (and its corresponding private key) to use.

### Accounts and Blocks

*   **Account:** A public Nano address (`nano_...` or `xrb_...`). Anyone can view its balance and history.
*   **Block:** A record of a transaction on the Nano ledger associated with a specific account (`send`, `receive`, `change`, `open`). Each block includes a hash of the previous block on that account's chain.
*   **Pending/Receivable:** Incoming funds (from a `send` block targeting an account) that haven't been accepted by a corresponding `receive` block created by the recipient account owner yet.

### **Sequential Operations & Concurrency**

*   **Nano Requirement:** The Nano protocol requires that blocks for a **single account** are added strictly **sequentially**. Each new block must reference the hash of the *previous* block on that account's chain. Trying to create multiple blocks simultaneously for the same account (e.g., two sends, or a send and a receive) will cause conflicts or errors (often called forks at the network level).
*   **API Enforcement:** To prevent these issues, this API implements **internal locking** for authenticated operations that modify the wallet state (`/wallet/send`, `/wallet/receive_all`, `/wallet/receive_one`, `/invoices/pay/*`).
*   **`429 Too Many Requests`:** If you attempt to start a modifying operation on an authenticated wallet while another one is already in progress *for that same wallet* (identified by the private key or seed/index), the API will immediately return an `HTTP 429 Too Many Requests` error.
*   **Client Handling:** Your application **must** be prepared to handle the `429` error when making authenticated calls. The recommended approach is to wait briefly and then retry the operation (potentially using an exponential backoff strategy). Read-only operations (`GET /wallet/*`) do not require locking and are not subject to the `429` error for concurrency reasons.

---

## 4. API Reference

*(For detailed Response/Request structures, see [Data Types (DTOs)](#data-types-dtos))*

### Health Check

*   `GET /`
*   **Description:** Checks if the API server is running and responsive.
*   **Auth:** None.
*   **Success:** `200 OK` with `{"status": "ok"}`.

### Public Account Information

*   **Applies to:** `/accounts/{account_address}/*` endpoints.
*   **Auth:** None.
*   **Path Param:** `account_address` (string) - The `nano_` or `xrb_` address.

Endpoints:

*   `GET /accounts/{account_address}/balance`
    *   **Description:** Get confirmed and pending balance for any account.
    *   **Success:** `200 OK` ([`AccountBalanceDto`](#accountbalancedto)).
    *   **Errors:** `400` (Bad Address Format), `503` (RPC Error).
*   `GET /accounts/{account_address}/history`
    *   **Description:** Get transaction history for any account.
    *   **Query Param:** `count` (int, optional, default: 50, max: 1000) - Number of entries.
    *   **Success:** `200 OK` ([`AccountHistoryDto`](#accounthistorydto)).
    *   **Errors:** `400` (Bad Address), `422` (Invalid Count), `503` (RPC Error).
*   `GET /accounts/{account_address}/info`
    *   **Description:** Get detailed ledger info for any account.
    *   **Success:** `200 OK` ([`AccountInfoDto`](#accountinfodto)).
    *   **Errors:** `400` (Bad Address), `503` (RPC Error).
*   `GET /accounts/{account_address}/receivable`
    *   **Description:** List pending incoming blocks for any account.
    *   **Query Param:** `threshold` (string, optional) - Minimum raw amount to list (e.g., `"1000000000000000000000000"`).
    *   **Success:** `200 OK` ([`AccountReceivableDto`](#accountreceivabledto)).
    *   **Errors:** `400` (Bad Address/Threshold Format), `422` (Invalid Threshold Value), `503` (RPC Error).

---

### Wallet Creation (Unauthenticated)

*   **Applies to:** `/wallet/create` endpoint.
*   **Auth:** None.

Endpoints:

*   `POST /wallet/create`
    *   **Description:** Generate a new Nano wallet seed, private key, and address.
    *   **Query Param:** `index` (int, optional, default: 0, min: 0) - The derivation index for the key pair.
    *   **Success:** `200 OK` ([`WalletCreationResponseDto`](#walletcreationresponsedto)).
    *   **Errors:** `422` (Invalid Index), `500` (Internal Server Error).
    *   **Security Note:** This endpoint generates wallet credentials entirely on the server side. While convenient, you may prefer to generate your seed locally using a secure random number generator for maximum security.

---

### Authenticated Wallet Operations

> **Reminder:** These endpoints REQUIRE `X-Wallet-Private-Key` **or** (`X-Wallet-Seed` + `X-Wallet-Index`) headers. See [Authentication](#authentication-critical-read).
> **Reminder:** Modifying operations (`POST`) **must be executed sequentially** for the same wallet. See [Sequential Operations & Concurrency](#sequential-operations--concurrency).

*   **Applies to:** `/wallet/*` endpoints. Operate on the single account derived from the provided key/seed/index.
*   **Auth:** Required (via Headers).

Endpoints:

*   `GET /wallet/balance`
    *   **Description:** Get *your* authenticated wallet balance. (Read-only, no 429 concurrency error).
    *   **Success:** `200 OK` ([`AccountBalanceDto`](#accountbalancedto)).
    *   **Errors:** `401` (Auth), `503` (RPC Error).
*   `GET /wallet/history`
    *   **Description:** Get *your* authenticated wallet history. (Read-only, no 429 concurrency error).
    *   **Query Param:** `count` (int, optional, default: 50, max: 1000).
    *   **Success:** `200 OK` ([`AccountHistoryDto`](#accounthistorydto)).
    *   **Errors:** `401` (Auth), `422` (Invalid Count), `503` (RPC Error).
*   `GET /wallet/info`
    *   **Description:** Get *your* authenticated wallet account info. (Read-only, no 429 concurrency error).
    *   **Success:** `200 OK` ([`AccountInfoDto`](#accountinfodto)).
    *   **Errors:** `401` (Auth), `503` (RPC Error).
*   `GET /wallet/receivable`
    *   **Description:** Get *your* authenticated wallet's pending blocks. (Read-only, no 429 concurrency error).
    *   **Query Param:** `threshold` (string, optional) - Min raw amount.
    *   **Success:** `200 OK` ([`AccountReceivableDto`](#accountreceivabledto)).
    *   **Errors:** `400` (Bad Threshold Format), `401` (Auth), `422` (Invalid Threshold Value), `503` (RPC Error).
*   `POST /wallet/send`
    *   **Description:** Send Nano from *your* authenticated wallet. Specify amount in **one** way: `amount_nano`, `amount_raw`, or (`nominal_amount` + `nominal_currency`). Providing nominal amount triggers calculation based on current exchange rates/depth. **This is a modifying operation subject to locking.**
    *   **Request Body:** [`SendPaymentRequestDto`](#sendpaymentrequestdto)
    *   **Success:** `200 OK` ([`PaymentResultDto`](#paymentresultdto)).
    *   **Errors:**
        *   `400` (Bad Request/Validation - e.g., invalid address, bad amount format, invalid nominal currency)
        *   `401` (Auth Error)
        *   `422` (Bad Body Structure/Invalid Input Value - e.g., negative amount)
        *   `429 Too Many Requests` (Another operation is already in progress for this wallet. Retry later.)
        *   `503` (RPC/Exchange Error, Insufficient Funds, Insufficient Market Depth for nominal conversion)

> **💙 Service Fee Information (for `/wallet/send`)**
>
> *   A small service fee (e.g., 0.5% - check server config) may be applied to the **/wallet/send** endpoint.
> *   **Calculation:** The fee is calculated on the final Nano amount determined for sending (whether specified directly or calculated from nominal).
> *   **Example:** If sending 1 XNO and fee is 0.5%, an additional 0.005 XNO is required.
> *   **Balance Requirement:** Your authenticated wallet **must have sufficient balance to cover BOTH the intended send amount AND the calculated service fee.** The total amount debited will be `intended_send_amount + fee_amount`. The `PaymentResultDto` response includes fee details.

*   `POST /wallet/receive_all`
    *   **Description:** Accept (receive) all pending funds for *your* authenticated wallet. Creates one `receive` block per pending `send`. **This is a modifying operation subject to locking.**
    *   **Success:** `200 OK` (`List[`[`ReceiveResultDto`](#receiveresultdto)`)`). Returns an empty list `[]` if no blocks were receivable.
    *   **Errors:**
        *   `401` (Auth Error)
        *   `429 Too Many Requests` (Another operation is already in progress for this wallet. Retry later.)
        *   `503` (RPC Error)
*   `POST /wallet/receive_one/{block_hash}`
    *   **Description:** Accept (receive) a *specific* pending block for *your* wallet. **This is a modifying operation subject to locking.**
    *   **Path Param:** `block_hash` (string) - The 64-character hex hash of the incoming `send` block.
    *   **Success:** `200 OK` ([`ReceiveResultDto`](#receiveresultdto)).
    *   **Errors:**
        *   `401` (Auth Error)
        *   `404` (Block hash not found or not receivable for this account)
        *   `429 Too Many Requests` (Another operation is already in progress for this wallet. Retry later.)
        *   `503` (RPC Error)

---

### Exchange Rate Information

*   **Applies to:** `/exchange/*` endpoints. Uses configured exchange adapter (e.g., Kraken).
*   **Auth:** None.

Endpoints:

*   `GET /exchange/rate/{quote_currency}`
    *   **Description:** Get current XNO exchange rate against a supported fiat currency.
    *   **Path Param:** `quote_currency` (string, e.g., `EUR`, `USD`). Case-insensitive.
    *   **Success:** `200 OK` ([`ExchangeRateResponse`](#exchangerateresponse)).
    *   **Errors:** `400` (Unsupported Pair), `503` (Exchange Service Error).
*   `GET /exchange/convert`
    *   **Description:** *Calculates* the result of converting between XNO and a supported fiat currency (or vice versa). Useful for display/information purposes before sending. Does *not* perform a transaction. One currency must be XNO.
    *   **Query Params:** `from_currency` (string), `to_currency` (string), `amount` (decimal, must be > 0). Case-insensitive.
    *   **Success:** `200 OK` ([`ConversionResponse`](#conversionresponse)).
    *   **Errors:** `400` (Bad Request/Validation - e.g., invalid currency, non-positive amount, pair not XNO<->Fiat), `422` (Missing/Invalid Query Params), `503` (Exchange Service Error, Insufficient Depth for calculation).

---

### External Invoices (SplitRoute)

*   **Applies to:** `/invoices/*` endpoints. Interacts with SplitRoute invoices.
*   **Auth:** Varies per endpoint.

Endpoints:

*   `GET /invoices/{invoice_id}`
    *   **Description:** Get details for a specific SplitRoute invoices.
    *   **Auth:** None (API uses its own provider key if configured).
    *   **Path Param:** `invoice_id` (string).
    *   **Success:** `200 OK` ([`InvoiceInfoDto`](#invoiceinfodto)).
    *   **Errors:** `400` (Bad ID Format), `404` (Invoice Not Found), `503` (External Provider API Error).
*   `POST /invoices/pay/{invoice_id}`
    *   **Description:** Pay a SplitRoute invoice using *your* authenticated Nano wallet. Fetches invoice details (amount/address) and attempts payment via `/wallet/send`. **This is a modifying operation subject to locking on the PAYING wallet.**
    *   **Auth:** Required (for the *paying* wallet, using `X-Wallet-...` headers).
    *   **Path Param:** `invoice_id` (string).
    *   **Success:** `200 OK` ([`InvoicePaymentResultDto`](#invoicepaymentresultdto)).
    *   **Errors:**
        *   `400` (Bad Invoice ID Format, Invoice Not Payable - e.g., expired, already paid)
        *   `401` (Auth Error for paying wallet)
        *   `404` (Invoice Not Found)
        *   `429 Too Many Requests` (Another operation is already in progress for the *paying* wallet. Retry later.)
        *   `503` (External Provider API Error, RPC Error, Insufficient Funds in paying wallet)

---

## 5. Data Types (DTOs)

*(Descriptions of the main JSON structures used. Refer to `src/application/dto/*.py` files for full field details, types, and validation rules.)*

*   **`AccountBalanceDto`**: Contains wallet balance. `balance_raw`, `pending_raw` (strings), `balance_nano`, `pending_nano` (decimals).
*   **`TransactionDto`**: Represents one block/transaction in history. Includes `type`, `account`, `amount_raw`/`nano`, `hash`, `timestamp`, `confirmed`, `link`, etc.
*   **`AccountHistoryDto`**: Contains `account` address and a `history` list of `TransactionDto`. May include `previous` hash for pagination (if implemented).
*   **`AccountInfoDto`**: Detailed ledger state: `frontier_block` hash, `open_block` hash, `balance_raw`/`nano`, `block_count`, `representative`, `modified_timestamp`, etc.
*   **`ReceivableBlockDto`**: Info about one pending block: `hash`, `amount_raw`/`nano`.
*   **`AccountReceivableDto`**: Contains a `blocks` dictionary mapping pending block hashes (string) to `ReceivableBlockDto`.
*   **`SendPaymentRequestDto`**: Used for `POST /wallet/send`. Requires `destination_address` and **exactly one** amount specification: `amount_nano` (decimal > 0), `amount_raw` (string, > "0"), or (`nominal_amount` (decimal > 0) + `nominal_currency` (string, e.g., "EUR", "USD")). Strict validation is applied.
*   **`PaymentResultDto`**: Response from successful `POST /wallet/send`. Includes `block_hash`, `sent_amount_raw`/`nano`, `destination_address`, plus `fee_amount_raw`/`nano` detailing the service fee applied (if any).
*   **`ReceiveResultDto`**: Response from successful `POST /wallet/receive_all` (list item) or `receive_one`. Includes `block_hash` (of the created receive block), `received_amount_raw`/`nano`, `source_address` (of the original send block), `confirmed` status.
*   **`ExchangeRateResponse`**: Response from `GET /exchange/rate/...`. Includes `base_currency` ("XNO"), `quote_currency`, `rate` (decimal).
*   **`ConversionResponse`**: Response from `GET /exchange/convert`. Includes `from_currency`, `to_currency`, `from_amount`, `to_amount`, `rate` (decimals).
*   **`InvoiceInfoDto`**: Response from `GET /invoices/{invoice_id}`. Details from external provider: `invoice_id`, `account_address`, `required_amount_raw`/`nano`, `status`, `expires_at`, `paid_at`, `nominal_amount`/`currency` (optional), `is_payable` (boolean indicating if it can be paid now).
*   **`InvoicePaymentResultDto`**: Response from successful `POST /invoices/pay/...`. Includes `invoice_id`, `nano_block_hash` of the payment transaction, `destination_address`, `amount_nano`/`raw` paid, `timestamp`.
*   **`WalletCreationResponseDto`**: Response from successful `POST /wallet/create`. Includes `seed` (64-character hex string), `index` (integer), `private_key` (64-character hex string), and `address` (nano_ prefixed address). **Handle this data securely as it gives full access to the wallet.**

---

## 6. Error Handling

The API uses standard HTTP status codes for signalling request outcomes. Always check the status code first. Error responses typically include a JSON body with a `detail` field explaining the issue.

*   **`200 OK`**: Success. The request was processed successfully.
*   **`400 Bad Request`**: General client-side error. The request was malformed or contained invalid data that prevents processing. Examples: invalid Nano address format, invalid threshold format, attempting to pay an unpayable invoice (e.g., expired). Check the `detail` field for specifics.
*   **`401 Unauthorized`**: Authentication failed. Missing, invalid, or incorrectly formatted `X-Wallet-Private-Key` or `X-Wallet-Seed`/`X-Wallet-Index` headers were provided for a protected endpoint.
*   **`404 Not Found`**: The requested resource does not exist. Examples: `GET /invoices/{non_existent_id}`, `POST /wallet/receive_one/{unknown_hash}`.
*   **`422 Unprocessable Entity`**: The request structure was syntactically correct JSON, but the *values* failed validation rules defined by the API (often in the DTOs). Examples: missing required fields in the request body, providing a negative amount, invalid enum value, providing multiple amount types in `/wallet/send`. The `detail` field usually contains a list of specific validation errors.
*   **`429 Too Many Requests`**: **Concurrency Limit Hit.** This status code is returned specifically for authenticated *modifying* operations (`POST /wallet/send`, `receive_all`, `receive_one`, `POST /invoices/pay/...`) when another such operation is already executing *for the same wallet*. This is due to the API enforcing sequential block creation required by Nano. **Clients MUST handle this by waiting and retrying the request later**, potentially using exponential backoff.
*   **`500 Internal Server Error`**: An unexpected error occurred on the server side that wasn't handled more specifically. This usually indicates a bug or configuration issue within the API itself. Check server logs.
*   **`503 Service Unavailable`**: A downstream dependency required to fulfill the request failed or is unavailable. This could be the Nano Node (RPC error), the configured exchange rate provider (e.g., Kraken API error), or SplitRoute API error. The `detail` field might provide more context, such as "Insufficient funds" reported by the Nano node during a send attempt, or "Exchange service error".

---

## 7. Use Cases & Examples

*(Examples reinforce security and potential 429 handling)*

**Use Case 1: Display Public Account Info**
Fetch `GET /accounts/{addr}/balance` and `GET /accounts/{addr}/history` to display account details on a public explorer or informational page. No authentication needed.

**Use Case 2: Create a New Nano Wallet**
1. Call `POST /wallet/create` to generate a new wallet seed, private key, and address.
2. Securely store the returned `seed` and/or `private_key` - this is the only time they will be available.
3. The returned `address` can be shared to receive funds.
4. Use the `seed` and `index` or `private_key` with authenticated endpoints when you need to send funds or check account information.

**Use Case 3: Send 0.5 XNO (Secure Backend Logic)**
1.  Obtain the destination `addr`.
2.  **Securely** retrieve the user's `private_key` or `seed`/`index` from a trusted store (e.g., encrypted database, environment variables on the server). **NEVER** get this from the user's browser directly for the API call.
3.  Make a `POST /wallet/send` request from your **backend server** including the correct authentication headers and a body like `{"destination_address": addr, "amount_nano": 0.5}`.
4.  **Handle the response:**
    *   `200 OK`: Payment sent successfully. Record the `block_hash` from the response.
    *   `429 Too Many Requests`: Another operation was in progress. Wait (e.g., 1-5 seconds) and retry the request. Implement a maximum retry limit.
    *   `4xx/5xx`: Handle other errors appropriately (log, inform user). Check `detail`.

**Use Case 4: Automated Fund Sweeping (Secure Script)**
1.  **Securely** load the `private_key` or `seed`/`index` of the account to sweep *within the script's secure environment*.
2.  Periodically:
    *   `POST /wallet/receive_all` (with auth headers). Handle potential `429` with retry.
    *   `GET /wallet/balance` (with auth headers).
    *   If `balance_raw` > "0":
        *   `POST /wallet/send` (with auth headers) using `amount_raw` set to the full `balance_raw` value, sending to a secure cold storage address. Handle potential `429` with retry. Handle insufficient funds errors (e.g., if a fee applies).
3.  Schedule script execution securely.

**Use Case 5: Pay an Invoice (Secure Backend Logic)**
1.  Get the `invoice_id` to be paid.
2.  `GET /invoices/{invoice_id}` to check status and `is_payable`. If not payable, stop.
3.  **Securely** retrieve the *payer's* `private_key` or `seed`/`index`.
4.  Make a `POST /invoices/pay/{invoice_id}` request from your **backend server** with the payer's auth headers.
5.  **Handle the response:**
    *   `200 OK`: Invoice paid successfully. Record the `nano_block_hash`.
    *   `429 Too Many Requests`: Another operation was in progress *for the payer's wallet*. Wait and retry.
    *   `4xx/5xx`: Handle other errors (invoice not found, not payable, insufficient funds, etc.). Check `detail`.

**(Code Examples - curl / Python)**

*Assume API at `http://localhost:8000` and **running on a secure backend**.*

**Get Public Balance (curl):**
```bash
curl http://localhost:8000/accounts/nano_3i1aq1cchnmbn9x5rsbap8b15akfh7wj7pwskuzi7ahz8oq6cobd99d4r3b7/balance
```

**Create New Wallet (curl):**
```bash
curl -X POST http://localhost:8000/wallet/create
```

Or with a specific index:
```bash
curl -X POST "http://localhost:8000/wallet/create?index=5"
```

Response:
```json
{
    "seed": "D73F1C31061762F139362B943CBB5E0C588F55A1EDB70EBEB31576D5CCA8E136",
    "index": 0,
    "private_key": "04C3335FDD43437956A57A21FD0EBC80A04F9AB8686CF4E5F1BC3DD543EFF3DC",
    "address": "nano_1q7h34aegsjrp94nmo1fcc8jz5jmr9ogj4wc78qy8saasog71e9q6dxqn3nc"
}
```

**Send 0.01 XNO via Private Key (curl - ⚠️ DANGEROUS if key exposed! Use only in secure backend scripts):**
```bash
# WARNING: EXTREME security risk if this key is exposed!
# Ensure this command runs ONLY in a secure, trusted server environment.
SECRET_NANO_KEY="YOUR_64_HEX_PRIVATE_KEY_HERE"
DEST_ADDR="nano_1..."
curl -X POST http://localhost:8000/wallet/send \
  -H "Content-Type: application/json" \
  -H "X-Wallet-Private-Key: ${SECRET_NANO_KEY}" \
  -d "{\"destination_address\": \"${DEST_ADDR}\", \"amount_nano\": 0.01}"
```

**Send 10 EUR worth of XNO via Seed/Index (curl - ⚠️ DANGEROUS if seed exposed! Use only in secure backend scripts):**
```bash
# WARNING: EXTREME security risk if this seed is exposed!
# Ensure this command runs ONLY in a secure, trusted server environment.
SECRET_NANO_SEED="YOUR_64_HEX_SEED_HERE"
WALLET_IDX="0"
DEST_ADDR="nano_1..."
curl -X POST http://localhost:8000/wallet/send \
  -H "Content-Type: application/json" \
  -H "X-Wallet-Seed: ${SECRET_NANO_SEED}" \
  -H "X-Wallet-Index: ${WALLET_IDX}" \
  -d "{\"destination_address\": \"${DEST_ADDR}\", \"nominal_amount\": 10.00, \"nominal_currency\": \"EUR\"}"
```

**Send Nano with Retry on 429 (Python - Backend Example):**
```python
import requests
import json
import time
import os
from decimal import Decimal

API_BASE_URL = "http://localhost:8000"
# WARNING: Load keys securely! Example uses environment variables.
WALLET_PRIVATE_KEY = os.environ.get("NANO_WALLET_PRIVATE_KEY")
DESTINATION_ADDRESS = "nano_1..."
AMOUNT_TO_SEND_NANO = Decimal("0.001")
MAX_RETRIES = 5
INITIAL_BACKOFF_S = 1.0

if not WALLET_PRIVATE_KEY:
    print("Error: NANO_WALLET_PRIVATE_KEY environment variable not set.")
    exit(1)

headers = {
    "Content-Type": "application/json",
    "X-Wallet-Private-Key": WALLET_PRIVATE_KEY
}
payload = {
    "destination_address": DESTINATION_ADDRESS,
    "amount_nano": float(AMOUNT_TO_SEND_NANO) # JSON needs float/str
}

retry_count = 0
backoff_time = INITIAL_BACKOFF_S

while retry_count < MAX_RETRIES:
    try:
        print(f"Attempt {retry_count + 1}: Sending {AMOUNT_TO_SEND_NANO} XNO...")
        response = requests.post(f"{API_BASE_URL}/wallet/send", headers=headers, json=payload, timeout=15)

        if response.status_code == 200:
            print("Success!")
            print(json.dumps(response.json(), indent=2))
            break # Exit loop on success
        elif response.status_code == 429:
            print(f"Received 429 Too Many Requests. Retrying in {backoff_time:.2f} seconds...")
            time.sleep(backoff_time)
            retry_count += 1
            backoff_time *= 2 # Exponential backoff
            # Optional: Add jitter to backoff_time
        else:
            # Handle other errors (4xx, 5xx)
            print(f"Error: {response.status_code}")
            try:
                print("Response:", response.json())
            except json.JSONDecodeError:
                print("Response:", response.text)
            break # Exit loop on non-429 error

    except requests.exceptions.RequestException as e:
        print(f"Network Error: {e}")
        # Decide if retry is appropriate for network errors
        print(f"Retrying in {backoff_time:.2f} seconds...")
        time.sleep(backoff_time)
        retry_count += 1
        backoff_time *= 2
    except Exception as e:
        print(f"Unexpected Python error: {e}")
        break # Exit on unexpected errors

if retry_count == MAX_RETRIES:
    print("Failed to send after maximum retries.")

```

**(Python example for paying an invoice would be similar, calling `POST /invoices/pay/{id}` and handling 429)**

---

## 8. Security Considerations

*   **🚨 KEY SECURITY IS PARAMOUNT 🚨:** The `X-Wallet-Private-Key` and `X-Wallet-Seed` headers ARE effectively your Nano private keys or seeds. **Their compromise means the IMMEDIATE AND IRREVERSIBLE LOSS OF FUNDS** associated with them. Protect these values with the highest level of security. Do NOT hardcode them in source control, log them, or expose them in any insecure manner. Use secure secret management practices (environment variables on secure servers, dedicated secret managers).
*   **🔒 BACKEND ONLY:** Authenticated API calls (`/wallet/*`, `/invoices/pay/*`) **MUST** originate exclusively from your secure, trusted backend server environment. **NEVER** make these calls directly from client-side code (JavaScript in browsers, mobile apps) as this inevitably exposes your keys/seeds.
*   **🔐 HTTPS/TLS:** Deploying this API behind HTTPS is **MANDATORY**. Failure to do so transmits your sensitive keys/seeds in plaintext over the network, making them trivial to intercept.
*   ** INPUT VALIDATION:** While the API performs validation, your application should also validate inputs (addresses, amounts) before sending them to the API as a best practice.
*   **🚦 RATE LIMITING (API Level):** Implement robust rate limiting on the API server itself (e.g., using middleware in FastAPI) to prevent denial-of-service attacks and general abuse, independent of the per-wallet concurrency locking.
*   **🔁 HANDLING CONCURRENCY (429):** Client applications making authenticated modifying calls **must** implement logic to handle the `429 Too Many Requests` error. This typically involves retrying the request after a short delay, potentially using exponential backoff and jitter to avoid thundering herd issues.
*   **🔥 ACCESS CONTROL:** Use firewalls, reverse proxies (like Nginx/Caddy), or cloud provider network security groups to restrict network access to the API server. Only allow connections from trusted IP addresses or networks (e.g., your application backend servers).
*   **AUDITING:** Consider adding logging (excluding sensitive headers!) to track authenticated operations for security monitoring.

---

## 9. Audit Logging and Chain Validation

The wallet application maintains a secure audit trail of all operations with cryptographic guarantees of integrity. 
This ensures that any unauthorized modifications to the log can be detected.

### Audit Log Features

- **Per-Account Sequence Numbers**: Each account maintains its own sequence of operations, starting from 1.
- **Content Hashing**: Each log entry includes a hash of its own content.
- **Continuous Chain**: The combination of sequence numbers and content hashing creates a verifiable audit chain per account.
- **Tamper Evidence**: Modifications to log entries or removal of entries can be detected with the validation tool.

### Audit Log Format

Each audit log entry is a JSON object with the following key fields:

```json
{
  "timestamp": "2023-06-15T12:34:56.789Z",
  "level": "INFO",
  "correlation_id": "123e4567-e89b-12d3-a456-426614174000",
  "event_type": "SUCCESS",
  "logger": "wallet_audit",
  "operation": "SEND_NANO",
  "wallet_address": "nano_address...",
  "account_sequence": 42,
  "request_details": { ... },
  "outcome_details": { ... },
  "entry_hash": "sha256_hash_value_of_this_entry"
}
```

Key fields for audit chain integrity:
- `wallet_address`: The account this operation applies to.
- `account_sequence`: Sequential number unique to this account.
- `entry_hash`: SHA-256 hash of the log entry content.


---