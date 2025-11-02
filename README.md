# ChainX Technology - HTTP 402 Payment Protocol

> **ChainX Protocol** - Solana-based HTTP 402 payment technology enabling programmatic payments for AI agents

ChainX Protocol is a production-ready HTTP 402 payment protocol built on Solana that enables autonomous programmatic payments with sub-second verification.

## 🚀 Quick Start

### What is ChainX Protocol?

ChainX Protocol implements the **HTTP 402 Payment Required** standard (RFC 7231) using blockchain payments on Solana. It allows servers to request payment before granting access to resources, enabling programmatic payments for AI agents and automation.

### Key Features

- ⚡ **Fast Verification** - Sub-second payment confirmation on Solana
- 💰 **Low Fees** - $0.00025 per transaction on Solana
- 🤖 **Programmatic** - Built for AI agents and automation
- 🔒 **Secure** - All payments verified on-chain
- 📡 **Standard HTTP** - Uses official HTTP 402 status code
- 🔄 **Automatic** - SDK handles entire payment flow

---

## 📖 How It Works

### HTTP 402 Flow

1. **Client makes request** → Server responds with `HTTP 402 Payment Required`
2. **Client reads payment headers** → Extracts payment amount, token, seller wallet
3. **Client creates payment** → Signs Solana transaction
4. **Client verifies payment** → With facilitator service
5. **Client retries request** → With payment proof headers
6. **Server verifies** → Grants access to resource

### Example Flow

```javascript
// Step 1: Initial request
const response = await fetch('https://api.chainx402.xyz/protected-data');
// Response: HTTP 402 Payment Required

// Step 2: Extract payment info
const paymentId = response.headers.get('X-Payment-Id');
const amount = response.headers.get('X-Payment-Amount');
const token = response.headers.get('X-Payment-Token');
const sellerWallet = response.headers.get('X-Payment-To');

// Step 3: Create and send payment
const signature = await sendSolanaPayment(sellerWallet, amount, token, paymentId);

// Step 4: Retry with payment proof
const paidResponse = await fetch('https://api.chainx402.xyz/protected-data', {
  headers: {
    'X-Payment-Id': paymentId,
    'X-Payment-Signature': signature
  }
});
// Response: HTTP 200 OK with data
```

---

## 🔌 API Endpoints

### Base URL

```
https://api.chainx402.xyz
```

### Facilitator Service

```
https://facilitator.chainx402.xyz
```

---

## 📡 HTTP 402 Headers

When a server responds with `HTTP 402 Payment Required`, it includes these headers:

| Header | Description | Example |
|--------|-------------|---------|
| `X-Payment-Required` | Indicates payment is required | `true` |
| `X-Payment-Id` | Unique payment identifier | `payment_abc123xyz` |
| `X-Payment-Amount` | Payment amount (decimal) | `0.0004` |
| `X-Payment-Token` | Token symbol | `USDC` |
| `X-Payment-Token-Mint` | Token contract address | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` |
| `X-Payment-To` | Seller wallet address | `YOUR_SELLER_WALLET` |
| `X-Payment-Memo` | Payment memo with ID | `ChainX:payment_abc123` |
| `X-Payment-Facilitator` | Facilitator service URL | `https://facilitator.chainx402.xyz` |

### Retry Headers

When retrying a request after payment, include:

| Header | Description | Example |
|--------|-------------|---------|
| `X-Payment-Id` | Original payment ID | `payment_abc123xyz` |
| `X-Payment-Signature` | Transaction signature | `5j7s8...` |

---

## 💻 Usage Examples

### JavaScript/TypeScript

```javascript
import { createAPIClient } from '@ChainX/buyer-sdk';

const client = createAPIClient({
  facilitatorUrl: 'https://facilitator.chainx402.xyz',
  rpcUrl: 'https://api.mainnet-beta.solana.com'
});

// Automatically handles HTTP 402, payment, and retry
const data = await client.get('https://api.chainx402.xyz', '/protected-data', {
  fromWallet: wallet.publicKey,
  signTransaction: wallet.signTransaction
});

console.log('Data:', data);
```

### Python

```python
import requests
from solana.transaction import Transaction
from solders.keypair import Keypair

def call_paid_api(endpoint):
    # Step 1: Initial request
    response = requests.get(f'https://api.chainx402.xyz{endpoint}')
    
    if response.status_code == 402:
        # Step 2: Extract payment info
        payment_id = response.headers.get('X-Payment-Id')
        amount = float(response.headers.get('X-Payment-Amount'))
        token = response.headers.get('X-Payment-Token')
        seller_wallet = response.headers.get('X-Payment-To')
        
        # Step 3: Create payment transaction
        signature = send_payment(seller_wallet, amount, token, payment_id)
        
        # Step 4: Verify payment
        verify_response = requests.post(
            'https://facilitator.chainx402.xyz/payment/verify',
            json={'paymentId': payment_id, 'signature': signature}
        )
        
        # Step 5: Retry with payment proof
        paid_response = requests.get(
            f'https://api.chainx402.xyz{endpoint}',
            headers={
                'X-Payment-Id': payment_id,
                'X-Payment-Signature': signature
            }
        )
        
        return paid_response.json()
    
    return response.json()
```

### cURL

```bash
# Step 1: Initial request (will get HTTP 402)
curl -i https://api.chainx402.xyz/protected-data

# Step 2: After payment, retry with headers
curl -H "X-Payment-Id: payment_abc123" \
     -H "X-Payment-Signature: 5j7s8..." \
     https://api.chainx402.xyz/protected-data
```

---

## 🏗️ For API Providers (Sellers)

### Set Up a Paid API Server

```typescript
import { createPaidAPIServer } from '@ChainX/seller-sdk';

const server = await createPaidAPIServer({
  facilitatorUrl: 'https://facilitator.chainx402.xyz',
  sellerWallet: 'YOUR_WALLET_ADDRESS',
  defaultToken: 'USDC',
  defaultTokenMint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v',
  pricePerRequest: 0.0004, // $0.0004 USDC
  port: 3000
});

// Protected endpoint
server.get('/api/data', paymentMiddleware, async (req, res) => {
  // This only executes after payment is verified
  return res.json({ data: 'Protected content here' });
});
```

### Payment Middleware

```typescript
import { PaymentMiddleware } from '@ChainX/seller-sdk';

const paymentMiddleware = new PaymentMiddleware({
  facilitatorUrl: 'https://facilitator.chainx402.xyz',
  sellerWallet: 'YOUR_WALLET_ADDRESS',
  defaultToken: 'USDC',
  pricePerRequest: 0.0004
});

// Use in your routes
app.get('/api/data', paymentMiddleware.createMiddleware({
  price: 0.0004,
  token: 'USDC'
}), async (req, res) => {
  // Protected resource
  res.json({ data: 'Protected data' });
});
```

---

## 🔧 Facilitator Service

### Payment Request

```http
POST https://facilitator.chainx402.xyz/payment/request
Content-Type: application/json

{
  "chain": "solana",
  "seller": "SELLER_WALLET_ADDRESS",
  "amount": 0.0004,
  "token": "USDC",
  "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "metadata": {
    "apiEndpoint": "/protected-data",
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

### Payment Verification

```http
POST https://facilitator.chainx402.xyz/payment/verify
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
  "timestamp": 1699123456789
}
```

---

## 💳 Payment Transaction (Solana)

### Create Payment Transaction

```javascript
import { Connection, PublicKey, Transaction } from '@solana/web3.js';
import { getAssociatedTokenAddress, createTransferInstruction } from '@solana/spl-token';

async function sendPayment(instructions, wallet) {
  const connection = new Connection('https://api.mainnet-beta.solana.com');
  
  // Get token accounts
  const tokenMint = new PublicKey(instructions.tokenMint);
  const sellerWallet = new PublicKey(instructions.to);
  const buyerTokenAccount = await getAssociatedTokenAddress(
    tokenMint, wallet.publicKey
  );
  const sellerTokenAccount = await getAssociatedTokenAddress(
    tokenMint, sellerWallet
  );
  
  // Create transfer instruction
  const transferInstruction = createTransferInstruction(
    buyerTokenAccount,
    sellerTokenAccount,
    wallet.publicKey,
    instructions.amount * 1e6, // USDC has 6 decimals
    [],
    TOKEN_PROGRAM_ID
  );
  
  // Create transaction
  const transaction = new Transaction().add(transferInstruction);
  transaction.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
  transaction.feePayer = wallet.publicKey;
  
  // Add memo
  transaction.add(new TransactionInstruction({
    keys: [],
    programId: new PublicKey('MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr'),
    data: Buffer.from(instructions.memo)
  }));
  
  // Sign and send
  const signed = await wallet.signTransaction(transaction);
  const signature = await connection.sendRawTransaction(signed.serialize());
  await connection.confirmTransaction(signature);
  
  return signature;
}
```

---

## 📚 SDKs

### Install SDKs

```bash
# Buyer SDK (for API consumers)
npm install @ChainX/buyer-sdk

# Seller SDK (for API providers)
npm install @ChainX/seller-sdk
```

### Buyer SDK Usage

```typescript
import { createAPIClient } from '@ChainX/buyer-sdk';

const client = createAPIClient({
  facilitatorUrl: 'https://facilitator.chainx402.xyz',
  rpcUrl: 'https://api.mainnet-beta.solana.com'
});

// GET request with auto-payment
const data = await client.get('https://api.chainx402.xyz', '/api/data', {
  fromWallet: wallet.publicKey,
  signTransaction: wallet.signTransaction
});

// POST request with auto-payment
const result = await client.post('https://api.chainx402.xyz', '/api/process', {
  fromWallet: wallet.publicKey,
  signTransaction: wallet.signTransaction,
  body: { input: 'data' }
});
```

### Seller SDK Usage

```typescript
import { PaymentMiddleware } from '@ChainX/seller-sdk';

const paymentMiddleware = new PaymentMiddleware({
  facilitatorUrl: 'https://facilitator.chainx402.xyz',
  sellerWallet: 'YOUR_WALLET_ADDRESS',
  defaultToken: 'USDC',
  pricePerRequest: 0.0004
});

// Express/Fastify middleware
app.get('/api/data', paymentMiddleware.createMiddleware({
  price: 0.0004,
  token: 'USDC'
}), async (req, res) => {
  // Payment verified, return data
  res.json({ data: 'Protected data' });
});
```

---

## 🌐 Supported Networks

### Solana

- **USDC** (SPL): `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`
- **SOL** (Native): Native Solana token
- **RPC Endpoints**: 
  - Mainnet: `https://api.mainnet-beta.solana.com`
  - Devnet: `https://api.devnet.solana.com`

---

## ⚡ Performance

| Metric | Value |
|--------|-------|
| **Payment Verification** | <400ms |
| **Transaction Fee** | ~$0.00025 |
| **Throughput** | 10,000+ tps |
| **Uptime** | 99.9% SLA |

---

## 🔒 Security

- ✅ All payments verified on-chain
- ✅ Transaction signatures validated
- ✅ Payment memos include unique payment IDs
- ✅ Facilitator service validates all transactions
- ✅ No sensitive data stored off-chain

---

## 📖 Resources

- **Documentation**: [https://chainx402.xyz/docs](https://chainx402.xyz/docs)
- **API Reference**: [https://chainx402.xyz/api](https://chainx402.xyz/api)
- **HTTP 402 Guide**: [https://chainx402.xyz/chainx402](https://chainx402.xyz/chainx402)
- **GitHub**: [https://github.com/Chainx402/chain](https://github.com/Chainx402/chain)

---

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

## 📄 License

MIT License - see [LICENSE](LICENSE) file for details.

---

## 📞 Support

- 📧 Email: support@chainx.xyz
- 💬 Discord: [Join our community](https://discord.gg/chainx)
- 🐛 Issues: [GitHub Issues](https://github.com/Chainx402/chainx-technology/issues)

---

**Built with ❤️ by the ChainX Protocol team**

