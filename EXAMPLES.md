# ChainX Protocol - Code Examples

Complete code examples for using ChainX Protocol in production at https://chainx402.xyz.

## JavaScript/TypeScript

### Using Buyer SDK

The buyer SDK provides a client for making requests to paid APIs. It automatically handles HTTP 402 responses, payment creation, and request retry.

```typescript
import { APIClient } from '@ChainX/buyer-sdk';

const client = new APIClient({
  facilitatorUrl: process.env.FACILITATOR_URL || 'https://chainx402.xyz/facilitator',
  walletConfig: {
    rpcUrl: 'https://api.mainnet-beta.solana.com',
    commitment: 'confirmed',
    maxRetries: 3
  }
});

// GET request with automatic payment handling
const result = await client.get('https://chainx402.xyz', '/api/data', {
  fromWallet: wallet.publicKey,
  signTransaction: wallet.signTransaction
});

if (result.success) {
  console.log('Data:', result.data);
  console.log('Payment signature:', result.paymentResult?.signature);
} else {
  console.error('Error:', result.error);
}
```

### Payment Client

The payment client handles Solana payment transactions.

```typescript
import { PaymentClient } from '@ChainX/buyer-sdk';

const paymentClient = new PaymentClient({
  rpcUrl: 'https://api.mainnet-beta.solana.com',
  commitment: 'confirmed',
  maxRetries: 3
});

// Make payment with instructions
const result = await paymentClient.makePayment(
  {
    paymentId: 'payment_abc123',
    amount: 0.0004,
    token: 'USDC',
    tokenMint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v',
    to: 'SELLER_WALLET_ADDRESS',
    memo: 'ChainX402:payment_abc123'
  },
  wallet.publicKey,
  wallet.signTransaction
);

if (result.success) {
  console.log('Payment signature:', result.signature);
} else {
  console.error('Payment failed:', result.error);
}
```

### Express.js Server with Payment

The seller SDK provides middleware for accepting payments in API endpoints.

```typescript
import express from 'express';
import { PaymentMiddleware } from '@ChainX/seller-sdk';

const app = express();
app.use(express.json());

const paymentMiddleware = new PaymentMiddleware({
  facilitatorUrl: process.env.FACILITATOR_URL || 'https://chainx402.xyz/facilitator',
  sellerWallet: process.env.SELLER_WALLET,
  defaultToken: 'USDC',
  defaultTokenMint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v',
  pricePerRequest: 0.0004,
  timeout: 300000 // 5 minutes
});

// Protected endpoint - middleware returns HTTP 402 if no payment
app.get('/api/data', 
  paymentMiddleware.createMiddleware({
    price: 0.0004,
    token: 'USDC'
  }),
  (req, res) => {
    // This only executes if payment is verified
    res.json({ 
      data: 'Protected content',
      paymentId: req.headers['x-payment-id']
    });
  }
);

app.listen(3000);
```

### Node.js/Fastify Server

```typescript
import { createPaidAPIServer } from '@ChainX/seller-sdk';

const server = await createPaidAPIServer({
  facilitatorUrl: process.env.FACILITATOR_URL || 'https://chainx402.xyz/facilitator',
  sellerWallet: process.env.SELLER_WALLET,
  defaultToken: 'USDC',
  defaultTokenMint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v',
  pricePerRequest: 0.0004,
  port: parseInt(process.env.PORT || '3000')
});

console.log('ChainX Protocol Paid API Server is running');
```

Available Endpoints:
- GET /health - Health check (free)
- GET /info - Service info (free)
- GET /api/data - Get data (paid - requires payment)
- POST /api/process - Process data (paid - requires payment)

## Solana Payment Implementation

### Payment Transaction Creation

```typescript
import { 
  Connection, 
  PublicKey, 
  Transaction, 
  SystemProgram,
  LAMPORTS_PER_SOL,
  sendAndConfirmTransaction,
  TransactionInstruction
} from '@solana/web3.js';
import { 
  getAssociatedTokenAddress, 
  createTransferInstruction
} from '@solana/spl-token';

class PaymentClient {
  private connection: Connection;

  async createPaymentTransaction(
    instructions: {
      token: 'USDC' | 'SOL';
      tokenMint?: string;
      amount: number;
      to: string;
      memo?: string;
    },
    fromWallet: PublicKey
  ): Promise<Transaction> {
    const transaction = new Transaction();

    if (instructions.token === 'SOL') {
      // SOL transfer
      transaction.add(
        SystemProgram.transfer({
          fromPubkey: fromWallet,
          toPubkey: new PublicKey(instructions.to),
          lamports: instructions.amount * LAMPORTS_PER_SOL
        })
      );
    } else {
      // SPL token transfer (USDC)
      const fromTokenAccount = await getAssociatedTokenAddress(
        new PublicKey(instructions.tokenMint!),
        fromWallet
      );
      const toTokenAccount = await getAssociatedTokenAddress(
        new PublicKey(instructions.tokenMint!),
        new PublicKey(instructions.to)
      );

      transaction.add(
        createTransferInstruction(
          fromTokenAccount,
          toTokenAccount,
          fromWallet,
          instructions.amount * Math.pow(10, 6) // USDC has 6 decimals
        )
      );
    }

    // Add memo instruction if provided
    if (instructions.memo) {
      transaction.add(
        new TransactionInstruction({
          keys: [],
          programId: new PublicKey('MemoSq4gqABAXKb96qnH8TysKcWfC85B2q2'),
          data: Buffer.from(instructions.memo, 'utf8')
        })
      );
    }

    return transaction;
  }
}

// Usage
const paymentClient = new PaymentClient();
const transaction = await paymentClient.createPaymentTransaction(
  {
    token: 'USDC',
    tokenMint: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v',
    amount: 0.0004,
    to: 'SELLER_WALLET_ADDRESS',
    memo: 'ChainX402:payment_abc123'
  },
  wallet.publicKey
);

const signed = await wallet.signTransaction(transaction);
const signature = await sendAndConfirmTransaction(
  connection,
  signed,
  [],
  { commitment: 'confirmed', maxRetries: 3 }
);
console.log('Payment signature:', signature);
```

## Python

### Basic Payment Flow

```python
import requests
from solana.transaction import Transaction
from solders.keypair import Keypair

def send_solana_payment(instructions, wallet):
    """Send Solana payment transaction"""
    connection = Connection("https://api.mainnet-beta.solana.com")
    
    # Get token accounts
    token_mint = Pubkey.from_string(instructions['tokenMint'])
    seller_wallet = Pubkey.from_string(instructions['to'])
    
    # Create transfer transaction
    # Implementation details...
    
    return signature

def call_paid_api(api_url, endpoint):
    """Call paid API with automatic payment handling"""
    # Step 1: Initial request
    response = requests.get(f'{api_url}{endpoint}')
    
    if response.status_code == 402:
        # Step 2: Extract payment info
        payment_id = response.headers.get('X-Payment-Id')
        amount = float(response.headers.get('X-Payment-Amount'))
        token = response.headers.get('X-Payment-Token')
        seller_wallet = response.headers.get('X-Payment-To')
        token_mint = response.headers.get('X-Payment-Token-Mint')
        
        # Step 3: Create payment transaction
        signature = send_solana_payment({
            'tokenMint': token_mint,
            'to': seller_wallet,
            'amount': amount,
            'memo': f'ChainX:{payment_id}'
        }, wallet)
        
        # Step 4: Retry with payment proof
        response = requests.get(
            f'{api_url}{endpoint}',
            headers={
                'X-Payment-Id': payment_id,
                'X-Payment-Signature': signature
            }
        )
    
    return response.json()

# Usage
data = call_paid_api('https://chainx402.xyz', '/api/data', payer_keypair)
print(data)
```

## Error Handling

```javascript
async function callPaidAPI(apiUrl, endpoint) {
  try {
    let response = await fetch(`${apiUrl}${endpoint}`);
    
    if (response.status === 402) {
      // Handle payment flow
      const paymentInfo = extractPaymentInfo(response);
      const signature = await processPayment(paymentInfo);
      
      // Retry with payment
      response = await fetch(`${apiUrl}${endpoint}`, {
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

For more information, visit: https://chainx402.xyz
