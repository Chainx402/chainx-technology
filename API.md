# ChainX Protocol - Complete API Reference

## Overview

ChainX Protocol provides a complete HTTP 402 payment implementation on Solana. This document covers all API endpoints, headers, and integration methods.

---

## Base URLs

### API Service
```
https://api.chainx402.xyz
```

### Facilitator Service
```
https://facilitator.chainx402.xyz
```

---

## HTTP 402 Payment Flow

### Step 1: Initial Request

```http
GET /protected-resource HTTP/1.1
Host: api.chainx402.xyz
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
X-Payment-Facilitator: https://facilitator.chainx402.xyz

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

### Step 2: Payment Request

```http
POST /payment/request HTTP/1.1
Host: facilitator.chainx402.xyz
Content-Type: application/json

{
  "chain": "solana",
  "seller": "SELLER_WALLET_ADDRESS",
  "amount": 0.0004,
  "token": "USDC",
  "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "metadata": {
    "apiEndpoint": "/protected-resource",
    "paymentId": "payment_abc123xyz",
    "timestamp": 1699123456789
  }
}
```

**Response:**
```json
{
  "paymentId": "payment_abc123xyz",
  "paymentInstructions": {
    "to": "SELLER_WALLET_ADDRESS",
    "amount": 0.0004,
    "token": "USDC",
    "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "memo": "ChainX:payment_abc123xyz"
  }
}
```

### Step 3: Payment Verification

```http
POST /payment/verify HTTP/1.1
Host: facilitator.chainx402.xyz
Content-Type: application/json

{
  "paymentId": "payment_abc123xyz",
  "signature": "5j7s8...",
  "chain": "solana"
}
```

**Response:**
```json
{
  "status": "verified",
  "paymentId": "payment_abc123xyz",
  "signature": "5j7s8...",
  "timestamp": 1699123456789,
  "amount": 0.0004,
  "token": "USDC"
}
```

### Step 4: Retry Request with Payment Proof

```http
GET /protected-resource HTTP/1.1
Host: api.chainx402.xyz
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

---

## HTTP Headers Reference

### Payment Request Headers (Server → Client)

| Header | Description | Type | Required |
|--------|-------------|------|----------|
| `X-Payment-Required` | Indicates payment is required | `string` | Yes |
| `X-Payment-Id` | Unique payment identifier | `string` | Yes |
| `X-Payment-Amount` | Payment amount (decimal) | `string` | Yes |
| `X-Payment-Token` | Token symbol | `string` | Yes |
| `X-Payment-Token-Mint` | Token contract/mint address | `string` | Yes |
| `X-Payment-To` | Seller wallet address | `string` | Yes |
| `X-Payment-Memo` | Payment memo with ID | `string` | Yes |
| `X-Payment-Facilitator` | Facilitator service URL | `string` | Yes |

### Payment Proof Headers (Client → Server)

| Header | Description | Type | Required |
|--------|-------------|------|----------|
| `X-Payment-Id` | Original payment ID | `string` | Yes |
| `X-Payment-Signature` | Transaction signature | `string` | Yes |

---

## Facilitator Service Endpoints

### Create Payment Request

**Endpoint:** `POST /payment/request`

**Request Body:**
```json
{
  "chain": "solana",
  "seller": "string",
  "amount": 0.0004,
  "token": "USDC",
  "tokenMint": "string",
  "metadata": {
    "apiEndpoint": "string",
    "paymentId": "string",
    "timestamp": 1699123456789
  }
}
```

**Response:**
```json
{
  "paymentId": "string",
  "paymentInstructions": {
    "to": "string",
    "amount": 0.0004,
    "token": "USDC",
    "tokenMint": "string",
    "memo": "string"
  }
}
```

### Verify Payment

**Endpoint:** `POST /payment/verify`

**Request Body:**
```json
{
  "paymentId": "string",
  "signature": "string",
  "chain": "solana"
}
```

**Response:**
```json
{
  "status": "verified",
  "paymentId": "string",
  "signature": "string",
  "timestamp": 1699123456789,
  "amount": 0.0004,
  "token": "USDC"
}
```

### Get Payment Status

**Endpoint:** `GET /payment/{paymentId}`

**Response:**
```json
{
  "status": "verified",
  "paymentId": "string",
  "signature": "string",
  "timestamp": 1699123456789,
  "amount": 0.0004,
  "token": "USDC"
}
```

---

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

---

## Code Examples

### Complete JavaScript Example

```javascript
async function callPaidAPI(endpoint) {
  // Step 1: Initial request
  let response = await fetch(`https://api.chainx402.xyz${endpoint}`);
  
  // Step 2: Check for HTTP 402
  if (response.status === 402) {
    // Extract payment info
    const paymentId = response.headers.get('X-Payment-Id');
    const amount = parseFloat(response.headers.get('X-Payment-Amount'));
    const token = response.headers.get('X-Payment-Token');
    const tokenMint = response.headers.get('X-Payment-Token-Mint');
    const sellerWallet = response.headers.get('X-Payment-To');
    const facilitatorUrl = response.headers.get('X-Payment-Facilitator');
    
    // Step 3: Get payment instructions
    const paymentRequest = await fetch(`${facilitatorUrl}/payment/request`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        chain: 'solana',
        seller: sellerWallet,
        amount: amount,
        token: token,
        tokenMint: tokenMint,
        metadata: {
          apiEndpoint: endpoint,
          paymentId: paymentId,
          timestamp: Date.now()
        }
      })
    });
    
    const { paymentInstructions } = await paymentRequest.json();
    
    // Step 4: Send payment transaction
    const signature = await sendSolanaPayment(paymentInstructions, wallet);
    
    // Step 5: Verify payment
    await fetch(`${facilitatorUrl}/payment/verify`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        paymentId: paymentId,
        signature: signature,
        chain: 'solana'
      })
    });
    
    // Step 6: Retry request with payment proof
    response = await fetch(`https://api.chainx402.xyz${endpoint}`, {
      headers: {
        'X-Payment-Id': paymentId,
        'X-Payment-Signature': signature
      }
    });
  }
  
  return await response.json();
}
```

---

## Rate Limits

- **API Requests**: 1000 requests per hour per IP
- **Payment Verification**: No limit
- **Payment Requests**: 100 requests per minute per wallet

---

## Best Practices

1. **Always verify payments** before granting access
2. **Cache payment verifications** to reduce facilitator calls
3. **Use payment IDs** to prevent duplicate charges
4. **Handle errors gracefully** - always check response status
5. **Store signatures** for audit trails
6. **Set payment timeouts** (recommended: 5 minutes)

---

For more information, visit: https://chainx402.xyz

