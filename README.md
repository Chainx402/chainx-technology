# ChainX Technology - ChainX Protocol

ChainX Protocol is a production-ready HTTP 402 implementation built on Solana for ChainX Protocol. This repository documents the real implementation running at https://chainx402.xyz.

## Production Implementation

ChainX Protocol is deployed and operational at:
- **Website**: https://chainx402.xyz
- **API Endpoint**: https://chainx402.xyz/api/data
- **Facilitator Service**: https://chainx402.xyz/facilitator

### Real Features

- On-chain Solana transaction verification
- ChainX Protocol HTTP 402 implementation
- Netlify serverless functions for scalability
- Real-time payment verification
- Production-ready error handling

## Quick Start

### What is ChainX Protocol?

ChainX Protocol implements HTTP 402 Payment Required for ChainX Protocol using Solana blockchain payments. It allows servers to request payment before granting access to resources, enabling programmatic payments for AI agents and automation.

### How It Works

1. Client makes request to `https://chainx402.xyz/api/data`
2. Server responds with HTTP 402 and payment headers:
   - `X-Payment-Id`: Unique payment identifier
   - `X-Payment-Amount`: 0.0004 USDC
   - `X-Payment-Token`: USDC
   - `X-Payment-To`: Seller wallet address
   - `X-Payment-Memo`: ChainX402:payment_id

3. Client creates Solana payment transaction:
   - Transfer 0.0004 USDC to seller wallet
   - Include memo instruction: `ChainX402:payment_id`
   - Get transaction signature

4. Client verifies payment:
   - POST to `https://chainx402.xyz/facilitator/payment/verify`
   - Body: `{ "paymentId": "...", "signature": "..." }`

5. Client retries original request with headers:
   - `X-Payment-Id`: payment_id
   - `X-Payment-Signature`: transaction_signature

6. Server verifies payment on-chain and returns data

## HTTP 402 Headers

When a server responds with HTTP 402 Payment Required, it includes these headers:

| Header | Description | Example |
|--------|-------------|---------|
| X-Payment-Required | Indicates payment is required | true |
| X-Payment-Id | Unique payment identifier | payment_abc123xyz |
| X-Payment-Amount | Payment amount (decimal) | 0.0004 |
| X-Payment-Token | Token symbol | USDC |
| X-Payment-Token-Mint | Token contract address | EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v |
| X-Payment-To | Seller wallet address | SELLER_WALLET_ADDRESS |
| X-Payment-Memo | Payment memo with ID | ChainX402:payment_abc123 |
| X-Payment-Facilitator | Facilitator service URL | https://chainx402.xyz/facilitator |

### Retry Headers

When retrying a request after payment, include:

| Header | Description | Example |
|--------|-------------|---------|
| X-Payment-Id | Original payment ID | payment_abc123xyz |
| X-Payment-Signature | Transaction signature | 5j7s8... |

## Production Implementation

### API Endpoints

**Free Endpoints (No Payment Required):**
- `GET https://chainx402.xyz/api/health` - Health check
- `GET https://chainx402.xyz/api/info` - Service information

**Paid Endpoints (HTTP 402 Payment Required):**
- `GET https://chainx402.xyz/api/data` - Get protected data
- `POST https://chainx402.xyz/api/process` - Process data

### Facilitator Service

The facilitator service verifies Solana transactions on-chain:

- `POST https://chainx402.xyz/facilitator/payment/request` - Create payment request
- `POST https://chainx402.xyz/facilitator/payment/verify` - Verify payment transaction
- `GET https://chainx402.xyz/facilitator/payment/:id` - Get payment status
- `GET https://chainx402.xyz/facilitator/health` - Health check

### Payment Verification

The facilitator service performs real on-chain verification:

1. Fetches transaction from Solana RPC
2. Validates transaction signature format
3. Checks for SPL token transfer (USDC) or SOL transfer
4. Verifies recipient matches seller wallet address
5. Verifies amount matches payment request (with tolerance for floating point)
6. Verifies token mint matches USDC (if SPL token)
7. Checks memo instruction contains payment ID
8. Returns verification result

## Code Examples

### JavaScript/TypeScript

```javascript
// Step 1: Make initial request
const response = await fetch('https://chainx402.xyz/api/data');

if (response.status === 402) {
  // Step 2: Extract payment info from headers
  const paymentId = response.headers.get('X-Payment-Id');
  const amount = parseFloat(response.headers.get('X-Payment-Amount'));
  const token = response.headers.get('X-Payment-Token');
  const sellerWallet = response.headers.get('X-Payment-To');
  const tokenMint = response.headers.get('X-Payment-Token-Mint');
  const memo = response.headers.get('X-Payment-Memo');
  
  // Step 3: Create Solana payment transaction
  // Use @solana/web3.js and @solana/spl-token
  const signature = await sendSolanaPayment({
    to: sellerWallet,
    amount: amount,
    tokenMint: tokenMint,
    memo: memo
  });
  
  // Step 4: Verify payment with facilitator
  const verifyResponse = await fetch(
    'https://chainx402.xyz/facilitator/payment/verify',
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ paymentId, signature })
    }
  );
  
  const verifyResult = await verifyResponse.json();
  
  if (verifyResult.status === 'verified') {
    // Step 5: Retry original request with payment proof
    const paidResponse = await fetch('https://chainx402.xyz/api/data', {
      headers: {
        'X-Payment-Id': paymentId,
        'X-Payment-Signature': signature
      }
    });
    
    const data = await paidResponse.json();
    console.log('Data:', data);
  }
}
```

### cURL Example

```bash
# Step 1: Request data (will get HTTP 402)
curl -i https://chainx402.xyz/api/data

# Step 2: Create payment transaction on Solana
# (Use Solana wallet to send 0.0004 USDC with memo)

# Step 3: Verify payment
curl -X POST https://chainx402.xyz/facilitator/payment/verify \
  -H "Content-Type: application/json" \
  -d '{"paymentId":"payment_id","signature":"transaction_signature"}'

# Step 4: Retry request with payment proof
curl -H "X-Payment-Id: payment_id" \
     -H "X-Payment-Signature: transaction_signature" \
     https://chainx402.xyz/api/data
```

## Solana Payment Transaction

The payment transaction must include:

- Token transfer instruction (0.0004 USDC to seller wallet)
- Memo instruction with: `ChainX402:payment_id`
- Valid transaction signature
- Confirmed on Solana mainnet

### Transaction Structure

```javascript
import { Transaction, TransactionInstruction } from '@solana/web3.js';
import { createTransferInstruction, getAssociatedTokenAddress } from '@solana/spl-token';

const transaction = new Transaction();

// Add USDC transfer instruction
const transferInstruction = createTransferInstruction(
  buyerTokenAccount,
  sellerTokenAccount,
  buyerWallet,
  amount * 1e6, // USDC has 6 decimals
  [],
  TOKEN_PROGRAM_ID
);
transaction.add(transferInstruction);

// Add memo instruction
transaction.add(new TransactionInstruction({
  keys: [],
  programId: new PublicKey('MemoSq4gqABAXKb96qnH8TysKcWfC85B2q2'),
  data: Buffer.from(`ChainX402:${paymentId}`)
}));

// Sign and send transaction
const signature = await connection.sendTransaction(transaction, [buyerWallet]);
await connection.confirmTransaction(signature);
```

## Supported Networks

### Solana Mainnet

- **USDC (SPL Token)**: `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`
- **SOL (Native Token)**: Native Solana token
- **RPC Endpoint**: `https://api.mainnet-beta.solana.com`

## Performance

| Metric | Value |
|--------|-------|
| Payment Verification | <400ms average |
| Transaction Fee | ~$0.00025 on Solana |
| API Response Time | <100ms |
| Uptime | Production-grade reliability |

## Testing

Complete testing documentation is available in [TESTING.md](./TESTING.md).

### Quick Test Commands

**Free Endpoints:**
```bash
# Health check
curl -i https://chainx402.xyz/api/health

# Service info
curl -i https://chainx402.xyz/api/info
```

**Paid Endpoint (ChainX Protocol HTTP 402):**
```bash
# Request protected data
curl -i https://chainx402.xyz/api/data

# Response: HTTP 402 with payment headers
# X-Payment-Id, X-Payment-Amount, X-Payment-Token, etc.
```

**Retry with Payment:**
```bash
curl -i https://chainx402.xyz/api/data \
  -H "X-Payment-Id: payment_id" \
  -H "X-Payment-Signature: transaction_signature"
```

**Facilitator Endpoints:**
```bash
# Service info
curl -i https://chainx402.xyz/facilitator

# Health check
curl -i https://chainx402.xyz/facilitator/health

# Create payment request
curl -i -X POST https://chainx402.xyz/facilitator/payment/request \
  -H "Content-Type: application/json" \
  -d '{"seller":"WALLET","amount":0.0004,"token":"USDC"}'

# Verify payment
curl -i -X POST https://chainx402.xyz/facilitator/payment/verify \
  -H "Content-Type: application/json" \
  -d '{"paymentId":"payment_id","signature":"tx_signature"}'
```

For complete testing examples, see [TESTING.md](./TESTING.md).

## Security

- All payments verified on-chain via Solana RPC
- Transaction signatures validated before acceptance
- Payment memos include unique payment IDs
- No sensitive data stored off-chain
- CORS configured for secure cross-origin requests

## Architecture

ChainX Protocol is deployed as Netlify serverless functions:

1. **API Function** (`/netlify/functions/api.js`):
   - Handles HTTP 402 responses
   - Processes payment verification
   - Returns protected data after verification

2. **Facilitator Function** (`/netlify/functions/facilitator.js`):
   - Creates payment requests
   - Verifies Solana transactions on-chain
   - Tracks payment status

Both functions use real Solana RPC connections for transaction verification.

## Resources

- **GitHub**: https://github.com/Chainx402/chainx-technology
- **X (Twitter)**: https://x.com/Chainx402
- **Website**: https://chainx402.xyz
- **Main Repository**: https://github.com/Chainx402/chain (private)

## Contributing

Contributions are welcome. Please feel free to submit a Pull Request.

## License

MIT License - see LICENSE file for details.

## Support

- X (Twitter): @Chainx402
- GitHub: chainx-technology
- Website: https://chainx402.xyz
