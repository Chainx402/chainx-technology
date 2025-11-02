# ChainX Protocol - Testing Guide

Complete testing guide for ChainX Protocol endpoints at https://chainx402.xyz.

## Quick Test Commands

### Free Endpoints

**Health Check:**
```bash
curl -i https://chainx402.xyz/api/health
```

**Expected Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "ok",
  "service": "ChainX Protocol API",
  "timestamp": 1762120867679
}
```

**Service Info:**
```bash
curl -i https://chainx402.xyz/api/info
```

**Expected Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "name": "ChainX Protocol API",
  "description": "ChainX Protocol HTTP 402 API",
  "version": "1.0.0",
  "pricing": {
    "perRequest": 0.0004,
    "token": "USDC"
  },
  "protocol": "ChainX Protocol"
}
```

### Paid Endpoints (ChainX Protocol HTTP 402)

**Request Protected Data:**
```bash
curl -i https://chainx402.xyz/api/data
```

**Expected Response (HTTP 402):**
```http
HTTP/1.1 402 Payment Required
Access-Control-Allow-Origin: *
Content-Type: application/json
X-Payment-Required: true
X-Payment-Id: payment_qhmu88zw9ns866pxglzbe
X-Payment-Amount: 0.0004
X-Payment-Token: USDC
X-Payment-Token-Mint: EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
X-Payment-To: SELLER_WALLET_ADDRESS
X-Payment-Memo: ChainX402:payment_qhmu88zw9ns866pxglzbe
X-Payment-Facilitator: https://chainx402.xyz/facilitator

{
  "error": "Payment Required",
  "code": 402,
  "payment": {
    "id": "payment_qhmu88zw9ns866pxglzbe",
    "amount": 0.0004,
    "token": "USDC",
    "instructions": {
      "amount": 0.0004,
      "token": "USDC",
      "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "to": "SELLER_WALLET_ADDRESS",
      "memo": "ChainX402:payment_qhmu88zw9ns866pxglzbe"
    }
  }
}
```

**Process Data Endpoint:**
```bash
curl -i -X POST https://chainx402.xyz/api/process \
  -H "Content-Type: application/json" \
  -d '{"data": "test"}'
```

**Expected Response:** HTTP 402 (same format as above)

### Retry with Payment Proof

After creating a Solana payment transaction, retry the request:

```bash
curl -i https://chainx402.xyz/api/data \
  -H "X-Payment-Id: payment_qhmu88zw9ns866pxglzbe" \
  -H "X-Payment-Signature: YOUR_TRANSACTION_SIGNATURE"
```

**Expected Response (HTTP 200):**
```json
{
  "payment": {
    "id": "payment_qhmu88zw9ns866pxglzbe",
    "status": "verified",
    "verifiedAt": 1762120867679
  },
  "service": {
    "name": "ChainX Protocol API",
    "version": "1.0.0",
    "endpoint": "/api/data",
    "url": "https://chainx402.xyz/api/data"
  },
  "timestamp": 1762120867679,
  "protocol": "ChainX Protocol",
  "network": "Solana"
}
```

## Facilitator Service Endpoints

### Service Info

```bash
curl -i https://chainx402.xyz/facilitator
```

**Expected Response:**
```json
{
  "service": "ChainX Protocol Facilitator",
  "version": "1.0.0",
  "description": "Solana payment verification service",
  "endpoints": {
    "health": "/facilitator/health",
    "createPayment": "/facilitator/payment/request",
    "verifyPayment": "/facilitator/payment/verify",
    "paymentStatus": "/facilitator/payment/:id"
  },
  "network": "Solana",
  "url": "https://chainx402.xyz/facilitator"
}
```

### Health Check

```bash
curl -i https://chainx402.xyz/facilitator/health
```

**Expected Response:**
```json
{
  "status": "healthy",
  "timestamp": 1762120867679,
  "service": "ChainX Protocol Facilitator",
  "version": "1.0.0"
}
```

### Create Payment Request

```bash
curl -i -X POST https://chainx402.xyz/facilitator/payment/request \
  -H "Content-Type: application/json" \
  -d '{
    "seller": "YOUR_WALLET_ADDRESS",
    "amount": 0.0004,
    "token": "USDC",
    "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
  }'
```

**Expected Response:**
```json
{
  "id": "payment_abc123xyz",
  "status": "pending",
  "paymentInstructions": {
    "amount": 0.0004,
    "token": "USDC",
    "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "to": "YOUR_WALLET_ADDRESS",
    "memo": "ChainX402:payment_abc123xyz"
  }
}
```

### Verify Payment

```bash
curl -i -X POST https://chainx402.xyz/facilitator/payment/verify \
  -H "Content-Type: application/json" \
  -d '{
    "paymentId": "payment_abc123xyz",
    "signature": "YOUR_TRANSACTION_SIGNATURE"
  }'
```

**Expected Response (Verified):**
```json
{
  "id": "payment_abc123xyz",
  "status": "verified",
  "transactionSignature": "YOUR_TRANSACTION_SIGNATURE",
  "verifiedAt": 1762120867679
}
```

**Expected Response (Failed):**
```json
{
  "error": "Payment verification failed"
}
```

### Get Payment Status

```bash
curl -i https://chainx402.xyz/facilitator/payment/payment_abc123xyz
```

**Expected Response:**
```json
{
  "id": "payment_abc123xyz",
  "status": "verified",
  "verifiedAt": 1762120867679,
  "transactionSignature": "YOUR_TRANSACTION_SIGNATURE"
}
```

## Test Results Summary

| Endpoint | Method | Expected Status | Description |
|----------|--------|-----------------|-------------|
| `/api/health` | GET | 200 OK | Health check endpoint |
| `/api/info` | GET | 200 OK | Service information |
| `/api/data` | GET | 402 Payment Required | Protected data endpoint |
| `/api/data` | GET (with payment) | 200 OK | Returns data after payment |
| `/api/process` | POST | 402 Payment Required | Data processing endpoint |
| `/api/process` | POST (with payment) | 200 OK | Processes data after payment |
| `/facilitator` | GET | 200 OK | Facilitator service info |
| `/facilitator/health` | GET | 200 OK | Facilitator health check |
| `/facilitator/payment/request` | POST | 200 OK | Create payment request |
| `/facilitator/payment/verify` | POST | 200 OK | Verify payment transaction |
| `/facilitator/payment/:id` | GET | 200 OK | Get payment status |

## Testing with JavaScript

```javascript
// Test health endpoint
const healthResponse = await fetch('https://chainx402.xyz/api/health');
const healthData = await healthResponse.json();
console.log('Health:', healthData);

// Test paid endpoint (will get HTTP 402)
const dataResponse = await fetch('https://chainx402.xyz/api/data');
console.log('Status:', dataResponse.status); // 402
console.log('Payment ID:', dataResponse.headers.get('X-Payment-Id'));
console.log('Amount:', dataResponse.headers.get('X-Payment-Amount'));

// After payment, retry with payment proof
const paidResponse = await fetch('https://chainx402.xyz/api/data', {
  headers: {
    'X-Payment-Id': paymentId,
    'X-Payment-Signature': transactionSignature
  }
});
const paidData = await paidResponse.json();
console.log('Data:', paidData);
```

## Testing with Python

```python
import requests

# Test health endpoint
health_response = requests.get('https://chainx402.xyz/api/health')
print('Health:', health_response.json())

# Test paid endpoint (will get HTTP 402)
data_response = requests.get('https://chainx402.xyz/api/data')
print('Status:', data_response.status_code)  # 402
print('Payment ID:', data_response.headers.get('X-Payment-Id'))
print('Amount:', data_response.headers.get('X-Payment-Amount'))

# After payment, retry with payment proof
paid_response = requests.get(
    'https://chainx402.xyz/api/data',
    headers={
        'X-Payment-Id': payment_id,
        'X-Payment-Signature': transaction_signature
    }
)
print('Data:', paid_response.json())
```

## Verification Checklist

- [ ] Health endpoint returns 200 OK
- [ ] Info endpoint returns service details
- [ ] Data endpoint returns HTTP 402 with payment headers
- [ ] All payment headers are present (X-Payment-Id, X-Payment-Amount, etc.)
- [ ] Facilitator endpoint returns service info
- [ ] Facilitator health check returns 200 OK
- [ ] Payment request creation returns payment instructions
- [ ] Payment verification returns status
- [ ] Retry with payment proof returns 200 OK

## Notes

- All endpoints use HTTPS
- CORS is enabled for all origins
- Payment verification requires on-chain Solana transaction
- Payment IDs are unique for each request
- Payment expires after 5 minutes (300 seconds)

