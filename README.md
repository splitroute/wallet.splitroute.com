# wallet.splitroute.com Nano Wallet API Documentation

This repository hosts the official documentation for the **wallet.splitroute.com Nano Wallet API**.

## Overview

The wallet.splitroute.com API provides a RESTful interface (HTTP/JSON) to interact with the Nano cryptocurrency network. It simplifies common wallet operations, allowing developers to programmatically:

*   Check balances and transaction history for any Nano account.
*   Send and receive Nano using an authenticated wallet.
*   Send Nano based on nominal fiat values (e.g., send $10 USD worth of Nano) (0.5% fee).
*   Interact with and pay SplitRoute invoices directly via the API for free.
*   Create new Nano wallets with generated seeds, private keys, and addresses.

**The primary, detailed API documentation can be found here:**

➡️ **[`overview.md`](./overview.md)** ⬅️

## API Endpoint

*   **Base URL:** `https://wallet.splitroute.com`

Key endpoint structures include:

*   `/accounts/{nano_address}/...` - Public information for any Nano account.
*   `/wallet/...` - Authenticated operations for a specific wallet (Requires secure handling of keys/seeds).
*   `/wallet/create` - Create a new Nano wallet (Non-authenticated endpoint).
*   `/invoices/...` - Interaction with SplitRoute invoices (fetching info, paying).
*   `/exchange/...` - Exchange rate information.

## ⚠️ Security Warning

**CRITICAL:** Authenticated endpoints (`/wallet/*`, `/invoices/pay/*`) require your **Nano Private Key** or **Seed + Index** to be sent directly in HTTP headers (`X-Wallet-Private-Key` or `X-Wallet-Seed` / `X-Wallet-Index`).

*   **These credentials grant FULL CONTROL over your funds.**
*   **NEVER expose these credentials in frontend code** (browsers, mobile apps). An attacker could easily steal them.
*   **ONLY use authenticated endpoints from a secure backend environment** that you fully control and trust.
*   **ALWAYS use HTTPS** to ensure credentials are encrypted during transmission.

Please read the [Authentication section in overview.md](./overview.md#authentication-critical-read) carefully before using authenticated endpoints.

## Getting Started

Please refer to the [Quick Start Guide](./overview.md#quick-start-for-developers) and the full [API Reference](./overview.md#api-reference) within [`overview.md`](./overview.md) for detailed usage instructions, data structures, error handling, and examples.