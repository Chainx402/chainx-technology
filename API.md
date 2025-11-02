# ChainX Protocol - API Reference

## Overview

ChainX Protocol provides HTTP 402 payment implementation on Solana. This document covers the real production API endpoints and payment flow at https://chainx402.xyz.

All endpoints and code examples are from the implementation in the repository.

## HTTP 402 Payment Flow

### Step 1: Initial Request

Client makes a request to a protected API endpoint.

**Request:**
```http
GET /api/data HTTP/1.1
Host: chainx402.xyz
```

**Response (HTTP 402):**
```http
HTTP/1.1 402 Payment Required
X-Payment-Required: true
X-Payment-Id: payment_abc123xyz
X-Payment-Amount: 0.0004
X-Payment-Token: USDC
X-Payment-Token-Mint: EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
X-Payment-To: SELLER_WALLET_ADDRESS
X-Payment-Memo: ChainX:payment_abc123xyz

{
  "error": "Payment Required",
  "code": 402,
  "payment": {
    "id": "payment_abc123xyz",
    "amount": 0.0004,
    "token": "USDC"
  }
}
```

### Step 2: Payment Transaction

Client creates and sends a Solana payment transaction with the payment details from the headers.

### Step 3: Retry Request with Payment Proof

Client retries the original request with payment proof headers.

**Request:**
```http
GET /api/data HTTP/1.1
Host: your-api-domain.com
X-Payment-Id: payment_abc123xyz
X-Payment-Signature: 5j7s8...
```

**Response (HTTP 200):**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": "Protected content here"
}
```

## HTTP Headers Reference

### Payment Request Headers (Server → Client)

When a server responds with HTTP 402 Payment Required, it includes these headers:

| Header | Description | Type | Required |
|--------|-------------|------|----------|
| X-Payment-Required | Indicates payment is required | string | Yes |
| X-Payment-Id | Unique payment identifier | string | Yes |
| X-Payment-Amount | Payment amount (decimal) | string | Yes |
| X-Payment-Token | Token symbol | string | Yes |
| X-Payment-Token-Mint | Token contract/mint address | string | Yes |
| X-Payment-To | Seller wallet address | string | Yes |
| X-Payment-Memo | Payment memo with ID | string | Yes |

### Payment Proof Headers (Client → Server)

When retrying a request after payment, include:

| Header | Description | Type | Required |
|--------|-------------|------|----------|
| X-Payment-Id | Original payment ID | string | Yes |
| X-Payment-Signature | Transaction signature | string | Yes |

## Payment Transaction Details

### Solana Transaction Structure

The payment transaction must include:

- Token transfer instruction (SOL or SPL token)
- Memo instruction with payment ID
- Valid signature from payer wallet
- Confirmed on-chain status

### Payment Verification

Payment verification checks:

1. Transaction signature is valid
2. Transaction is confirmed on-chain
3. Recipient matches seller wallet address
4. Amount matches payment request
5. Token mint matches (if SPL token)
6. Memo includes payment ID

## Error Responses

### HTTP 400 - Bad Request

```json
{
  "error": "Bad Request",
  "message": "Invalid payment request"
}
```

### HTTP 402 - Payment Required

```json
{
  "error": "Payment Required",
  "code": 402,
  "payment": {
    "id": "payment_abc123",
    "amount": 0.0004,
    "token": "USDC"
  }
}
```

### HTTP 402 - Payment Verification Failed

```json
{
  "error": "Payment verification failed",
  "code": 402
}
```

### HTTP 429 - Rate Limit Exceeded

```json
{
  "error": "Rate limit exceeded",
  "retryAfter": 60
}
```

### HTTP 500 - Internal Server Error

```json
{
  "error": "Internal server error",
  "message": "Payment processing failed"
}
```

## Best Practices

1. Always verify payments before granting access
2. Cache payment verifications to reduce RPC calls
3. Use payment IDs to prevent duplicate charges
4. Handle errors gracefully - always check response status
5. Store signatures for audit trails
6. Set payment timeouts (recommended: 5 minutes)

For more information, visit: https://chainx402.xyz
