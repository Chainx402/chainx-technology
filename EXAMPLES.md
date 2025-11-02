# ChainX Protocol - Code Examples

Complete code examples for using ChainX Protocol in various languages and frameworks.

---

## JavaScript/TypeScript

### Basic Payment Flow

```javascript
async function callPaidAPI(apiUrl, endpoint) {
  // Step 1: Initial request
  let response = await fetch(`${apiUrl}${endpoint}`);
  
  if (response.status === 402) {
    // Step 2: Extract payment information
    const paymentId = response.headers.get('X-Payment-Id');
    const amount = parseFloat(response.headers.get('X-Payment-Amount'));
    const token = response.headers.get('X-Payment-Token');
    const sellerWallet = response.headers.get('X-Payment-To');
    const facilitatorUrl = response.headers.get('X-Payment-Facilitator');
    
    // Step 3: Get payment instructions from facilitator
    const paymentRequest = await fetch(`${facilitatorUrl}/payment/request`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        chain: 'solana',
        seller: sellerWallet,
        amount: amount,
        token: token,
        tokenMint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v',
        metadata: {
          apiEndpoint: endpoint,
          paymentId: paymentId,
          timestamp: Date.now()
        }
      })
    });
    
    const { paymentInstructions } = await paymentRequest.json();
    
    // Step 4: Send Solana payment
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
    
    // Step 6: Retry with payment proof
    response = await fetch(`${apiUrl}${endpoint}`, {
      headers: {
        'X-Payment-Id': paymentId,
        'X-Payment-Signature': signature
      }
    });
  }
  
  return await response.json();
}

// Usage
const data = await callPaidAPI('https://api.chainx402.xyz', '/api/data');
console.log(data);
```

### Using Buyer SDK

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
  body: { input: 'test data' }
});
```

### Express.js Server with Payment

```javascript
const express = require('express');
const { PaymentMiddleware } = require('@ChainX/seller-sdk');

const app = express();
const paymentMiddleware = new PaymentMiddleware({
  facilitatorUrl: 'https://facilitator.chainx402.xyz',
  sellerWallet: 'YOUR_WALLET_ADDRESS',
  defaultToken: 'USDC',
  pricePerRequest: 0.0004
});

// Protected endpoint
app.get('/api/data', paymentMiddleware.createMiddleware({
  price: 0.0004,
  token: 'USDC'
}), (req, res) => {
  // Payment verified, return data
  res.json({ data: 'Protected content' });
});

app.listen(3000);
```

---

## Python

### Basic Payment Flow

```python
import requests
from solana.transaction import Transaction
from solders.keypair import Keypair
from solders.pubkey import Pubkey
from spl.token.instructions import transfer, TransferParams
from spl.token.client import Token

def send_solana_payment(instructions, wallet):
    """Send Solana payment transaction"""
    connection = Connection("https://api.mainnet-beta.solana.com")
    
    # Get token accounts
    token_mint = Pubkey.from_string(instructions['tokenMint'])
    seller_wallet = Pubkey.from_string(instructions['to'])
    
    # Create transfer transaction
    # ... (implementation details)
    
    return signature

def call_paid_api(endpoint):
    """Call paid API with automatic payment handling"""
    api_url = 'https://api.chainx402.xyz'
    facilitator_url = 'https://facilitator.chainx402.xyz'
    
    # Step 1: Initial request
    response = requests.get(f'{api_url}{endpoint}')
    
    if response.status_code == 402:
        # Step 2: Extract payment info
        payment_id = response.headers.get('X-Payment-Id')
        amount = float(response.headers.get('X-Payment-Amount'))
        token = response.headers.get('X-Payment-Token')
        seller_wallet = response.headers.get('X-Payment-To')
        token_mint = response.headers.get('X-Payment-Token-Mint')
        
        # Step 3: Get payment instructions
        payment_request = requests.post(
            f'{facilitator_url}/payment/request',
            json={
                'chain': 'solana',
                'seller': seller_wallet,
                'amount': amount,
                'token': token,
                'tokenMint': token_mint,
                'metadata': {
                    'apiEndpoint': endpoint,
                    'paymentId': payment_id,
                    'timestamp': int(time.time() * 1000)
                }
            }
        )
        
        payment_instructions = payment_request.json()['paymentInstructions']
        
        # Step 4: Send payment
        signature = send_solana_payment(payment_instructions, wallet)
        
        # Step 5: Verify payment
        verify_response = requests.post(
            f'{facilitator_url}/payment/verify',
            json={
                'paymentId': payment_id,
                'signature': signature,
                'chain': 'solana'
            }
        )
        
        # Step 6: Retry with payment proof
        response = requests.get(
            f'{api_url}{endpoint}',
            headers={
                'X-Payment-Id': payment_id,
                'X-Payment-Signature': signature
            }
        )
    
    return response.json()

# Usage
data = call_paid_api('/api/data')
print(data)
```

### FastAPI Server with Payment

```python
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse
from chainx.seller_sdk import PaymentMiddleware

app = FastAPI()

payment_middleware = PaymentMiddleware(
    facilitator_url='https://facilitator.chainx402.xyz',
    seller_wallet='YOUR_WALLET_ADDRESS',
    default_token='USDC',
    price_per_request=0.0004
)

@app.get("/api/data")
async def get_data(request: Request):
    # Check payment headers
    payment_id = request.headers.get('X-Payment-Id')
    signature = request.headers.get('X-Payment-Signature')
    
    if not payment_id or not signature:
        # Return HTTP 402
        return JSONResponse(
            status_code=402,
            content={"error": "Payment Required"},
            headers={
                "X-Payment-Id": payment_middleware.create_payment_id(),
                "X-Payment-Amount": "0.0004",
                "X-Payment-Token": "USDC",
                # ... other headers
            }
        )
    
    # Verify payment
    is_valid = await payment_middleware.verify_payment(payment_id, signature)
    
    if not is_valid:
        return JSONResponse(
            status_code=402,
            content={"error": "Payment verification failed"}
        )
    
    # Payment verified, return data
    return {"data": "Protected content"}
```

---

## Node.js/Express

### Complete Server Example

```javascript
const express = require('express');
const { PaymentMiddleware } = require('@ChainX/seller-sdk');

const app = express();
app.use(express.json());

const paymentMiddleware = new PaymentMiddleware({
  facilitatorUrl: 'https://facilitator.chainx402.xyz',
  sellerWallet: 'YOUR_WALLET_ADDRESS',
  defaultToken: 'USDC',
  pricePerRequest: 0.0004
});

// Health check (free)
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

// Protected endpoint
app.get('/api/data', 
  paymentMiddleware.createMiddleware({
    price: 0.0004,
    token: 'USDC'
  }),
  (req, res) => {
    res.json({
      data: 'Protected data',
      paymentId: req.headers['x-payment-id']
    });
  }
);

// Protected POST endpoint
app.post('/api/process',
  paymentMiddleware.createMiddleware({
    price: 0.0004,
    token: 'USDC'
  }),
  (req, res) => {
    const { input } = req.body;
    res.json({
      result: `Processed: ${input}`,
      paymentId: req.headers['x-payment-id']
    });
  }
);

app.listen(3000, () => {
  console.log('ChainX Protocol API Server running on port 3000');
});
```

---

## Solana Payment Implementation

### Complete Solana Payment Example

```javascript
import { Connection, PublicKey, Transaction } from '@solana/web3.js';
import { 
  getAssociatedTokenAddress, 
  createTransferInstruction,
  TOKEN_PROGRAM_ID
} from '@solana/spl-token';
import { createMemoInstruction } from '@solana/web3.js';

async function sendSolanaPayment(instructions, wallet, connection) {
  const tokenMint = new PublicKey(instructions.tokenMint);
  const sellerWallet = new PublicKey(instructions.to);
  const buyerWallet = wallet.publicKey;
  
  // Get token accounts
  const buyerTokenAccount = await getAssociatedTokenAddress(
    tokenMint,
    buyerWallet
  );
  const sellerTokenAccount = await getAssociatedTokenAddress(
    tokenMint,
    sellerWallet
  );
  
  // Create transfer instruction
  const transferInstruction = createTransferInstruction(
    buyerTokenAccount,
    sellerTokenAccount,
    buyerWallet,
    instructions.amount * 1e6, // USDC has 6 decimals
    [],
    TOKEN_PROGRAM_ID
  );
  
  // Create transaction
  const transaction = new Transaction().add(transferInstruction);
  
  // Add memo
  const memoInstruction = createMemoInstruction(
    instructions.memo,
    [buyerWallet]
  );
  transaction.add(memoInstruction);
  
  // Get recent blockhash
  const { blockhash } = await connection.getLatestBlockhash();
  transaction.recentBlockhash = blockhash;
  transaction.feePayer = buyerWallet;
  
  // Sign transaction
  const signed = await wallet.signTransaction(transaction);
  
  // Send transaction
  const signature = await connection.sendRawTransaction(signed.serialize());
  
  // Confirm transaction
  await connection.confirmTransaction(signature, 'confirmed');
  
  return signature;
}

// Usage
const signature = await sendSolanaPayment(
  paymentInstructions,
  wallet,
  connection
);
console.log('Payment sent:', signature);
```

---

## React/Next.js Client Example

```jsx
import { useState } from 'react';
import { useWallet } from '@solana/wallet-adapter-react';
import { Connection, PublicKey } from '@solana/web3.js';

export default function ChainXClient() {
  const { publicKey, signTransaction } = useWallet();
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  
  async function fetchPaidData() {
    setLoading(true);
    try {
      const client = createAPIClient({
        facilitatorUrl: 'https://facilitator.chainx402.xyz',
        rpcUrl: 'https://api.mainnet-beta.solana.com'
      });
      
      const result = await client.get(
        'https://api.chainx402.xyz',
        '/api/data',
        {
          fromWallet: publicKey,
          signTransaction: signTransaction
        }
      );
      
      setData(result);
    } catch (error) {
      console.error('Error:', error);
    } finally {
      setLoading(false);
    }
  }
  
  return (
    <div>
      <button onClick={fetchPaidData} disabled={loading}>
        {loading ? 'Loading...' : 'Fetch Paid Data'}
      </button>
      {data && <pre>{JSON.stringify(data, null, 2)}</pre>}
    </div>
  );
}
```

---

## cURL Examples

### Basic Request

```bash
# Initial request (will get HTTP 402)
curl -i https://api.chainx402.xyz/api/data

# After payment, retry with headers
curl -H "X-Payment-Id: payment_abc123" \
     -H "X-Payment-Signature: 5j7s8..." \
     https://api.chainx402.xyz/api/data
```

### Payment Request

```bash
curl -X POST https://facilitator.chainx402.xyz/payment/request \
  -H "Content-Type: application/json" \
  -d '{
    "chain": "solana",
    "seller": "SELLER_WALLET",
    "amount": 0.0004,
    "token": "USDC",
    "tokenMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
  }'
```

### Payment Verification

```bash
curl -X POST https://facilitator.chainx402.xyz/payment/verify \
  -H "Content-Type: application/json" \
  -d '{
    "paymentId": "payment_abc123",
    "signature": "5j7s8...",
    "chain": "solana"
  }'
```

---

## Error Handling

```javascript
async function callPaidAPI(endpoint) {
  try {
    let response = await fetch(`https://api.chainx402.xyz${endpoint}`);
    
    if (response.status === 402) {
      // Handle payment flow
      const paymentInfo = extractPaymentInfo(response);
      const signature = await processPayment(paymentInfo);
      
      // Retry with payment
      response = await fetch(`https://api.chainx402.xyz${endpoint}`, {
        headers: {
          'X-Payment-Id': paymentInfo.paymentId,
          'X-Payment-Signature': signature
        }
      });
    }
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    
    return await response.json();
  } catch (error) {
    console.error('API call failed:', error);
    throw error;
  }
}
```

---

For more examples, visit: https://chainx402.xyz/examples

